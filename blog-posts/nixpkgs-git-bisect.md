---
tags:
  - nixos
  - linux
  - debugging
  - git
title: hunting a nixpkgs regression with git bisect
summary: using git bisect to hunt down a problematic change in nixpkgs
date: 2025-04-03
publish: yes
---
recently i ran into an issue where i updated my nix flake inputs, using the latest version of nixpkgs unstable to build my nixos configuration, and it showed an error when trying to build my configuration:

    error: path '/nix/store/i0wfxny3k1sl4w9ldgsi9f1ww3kq0369-linux-6.12.19-modules-shrunk/lib' is not in the Nix store

initially i was thinking it might be an issue with my configuration but after removing all configuration and building the most minimal nixos possible it still wasn't working. i realised it was likely an issue with nixpkgs itself, so i decided to try and use `git bisect` to figure out where the regression was introduced.

`git bisect` helps you perform a binary search on all commits between a provided known working commit and a known broken one, which seemed ideal for this situation.

first i needed to create an environment that would easily let me test each revision. to do this, i ran the version of nix i was using in a podman container:

    ❯ podman run -it nixos/nix:2.18.5 sh

from some previous manual testing, i had already found a commit from the past that is good, then a more recent one that is bad. with these two hashes, i started bisecting:

    ❯ git bisect start --first-parent bcc57092e37f2623f701ab3afb8a853da48441fa 67a6eb4d6c145ebcd01deeaf3d88d587c3458763
    Updating files: 100% (9738/9738), done.
    Previous HEAD position was 38a133a96601 chromaprint: add nix-update-script
    Switched to branch 'master'
    Your branch is up to date with 'origin/master'.
    Bisecting: 728 revisions left to test after this (roughly 10 steps)
    [2989a2383a89af2bfd77ecca617fdf0c9d5296f8] xdg-terminal-exec: 0.12.2 -> 0.12.3 (#392241)

sidenote: when bisecting nixpkgs it's important to use `--first-parent` so you wont be testing out commits from eg pull requests that wont have any build artifacts cached, meaning you'll have to build the whole system from source.

then i created a flake.nix with a very minimal nixosConfiguration, using the revision of nixpkgs provided by `git bisect`:

    :::nix
    {
      inputs = {
        nixpkgs.url = "github:NixOS/nixpkgs/2989a2383a89af2bfd77ecca617fdf0c9d5296f8";
      };
    
      outputs = {nixpkgs, ...}: {
        nixosConfigurations.minimal = nixpkgs.lib.nixosSystem {
          system = "x86_64-linux";
          modules = [
            {
              boot.loader.grub.enable = false;
              fileSystems."/" = {
                device = "none";
                fsType = "tmpfs";
              };
              system.stateVersion = "25.05";
            }
          ];
        };
      };
    }

now i could build the system for that specific revision of nixpkgs and see if we trigger the error:

    ❯ nix build .#nixosConfigurations.minimal.config.system.build.toplevel --extra-experimental-features nix-command --extra-experimental-features flakes
    warning: creating lock file '/root/flake.lock'
    error: path '/nix/store/i0wfxny3k1sl4w9ldgsi9f1ww3kq0369-linux-6.12.19-modules-shrunk/lib' is not in the Nix store

we did! so now lets mark that revision as bad and do it all again with the next revision in the bisect

    nixpkgs on  HEAD (2989a23) (BISECTING) took 2s
    ❯ git bisect bad
    Bisecting: 363 revisions left to test after this (roughly 9 steps)
    [fa88185d763e4cb0eee38b06da566f1ad16db30a] nats-server: 2.10.26 -> 2.11.0 (#391437)

after doing this for some time, we'll eventually find a culprit commit

    nixpkgs on  HEAD (fa88185) (BISECTING)
    ❯ git bisect good
    Bisecting: 181 revisions left to test after this (roughly 8 steps)
    [3e8f4560cf2ec20c07eacca7693486ef8532df78] python312Packages.types-awscrt: 0.24.1 -> 0.24.2 (#392177)
    
    nixpkgs on  HEAD (3e8f456) (BISECTING)
    ❯ git bisect bad
    Bisecting: 90 revisions left to test after this (roughly 7 steps)
    [624f7d949adc782408058437768496d472a1f258] gose: 0.9.0 -> 0.10.2 (#391264)
    
    [... cut for brevity ...]
    
    nixpkgs on  HEAD (6dd77b6) (BISECTING)
    ❯ git bisect good
    Bisecting: 0 revisions left to test after this (roughly 1 step)
    [8338df11c2a8fe804f5d0367f67533279357109a] python312Packages.smolagents: disable test that requires missing dep `mlx-lm` (#391629)
    
    nixpkgs on  HEAD (8338df1) (BISECTING)
    ❯ git bisect good
    3fcae17eabac8fdc6599d1c67d89726af3682613 is the first bad commit
    commit 3fcae17eabac8fdc6599d1c67d89726af3682613
    Merge: 8338df11c2a8 7233659eafee
    Author: Vladimír Čunát <v@cunat.cz>
    Date:   Sat Mar 22 17:39:24 2025 +0100
    
        staging-next 2025-03-13 (#389579)
    
     nixos/doc/manual/release-notes/rl-2505.section.md                      |   3 +
     nixos/modules/profiles/installation-device.nix                         |   4 +-
     pkgs/applications/audio/cdparanoia/default.nix                         |  90 ++++++-
     pkgs/applications/editors/emacs/build-support/generic.nix              |   1 +
     pkgs/applications/misc/sl1-to-photon/default.nix                       |   1 -
     pkgs/applications/version-management/sourcehut/core.nix                |   1 -

it's a [staging-next merge](https://github.com/NixOS/nixpkgs/pull/389579/files), which is unfortunate because this one commit contains many changes merged together.

searching for the word "store" eventually brought me to two specific files relating to `kernel/make-initrd` which seemed promising as the error was complaining about not being able to find specific kernel modules in the nix store. looking at the commits, i found the [original pull request](https://github.com/NixOS/nixpkgs/pull/372931) that made the change.  

at this point all i had to do was build from a commit just before the change and just after, which confirmed that this was indeed the breaking change. however, after some discussion on the [nixos discourse](https://discourse.nixos.org/t/issue-building-linux-kernel-modules-after-flake-update/62322/8?u=samiser) it was pointed out that `2.18.5` was actually an unsupported version, and the bug was not present on supported versions of nix (like `2.18.9`). 

after updating the version of nix on my server, i was indeed able to build the configuration just fine. i still thought this was a pretty interesting exercise in hunting down a problem though, and i hope you enjoyed reading about it :)
