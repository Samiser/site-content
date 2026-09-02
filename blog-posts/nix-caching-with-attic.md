---
tags:
    - nixos
    - nix
    - ci
    - self-hosting
title: caching nix CI builds with attic
summary: how github actions builds my nix configs and pushes them to a self-hosted attic cache
date: 2026-09-02
publish: yes
---

i manage all of my machines in a
[monorepo with a single flake](https://github.com/Samiser/nix-configs), and one
annoyance was building the same things repeatedly. every host builds a lot of
similar stuff, and CI was rebuilding all of it from scratch on every push. the
standard fix for this is a binary cache, so builds happen once and everything
else just downloads the results.

one solution would be [cachix](https://www.cachix.org/), but i like self-hosting
things, so instead i run [attic](https://github.com/zhaofengli/attic), a
self-hostable nix binary cache server. this post goes over how the cache is set
up and how github actions builds and pushes to it.

also worth mentioning i used to use [garnix](https://garnix.io/) (RIP) for most
of this stuff, so this is also kind of documenting my effort to replace garnix
for personal use since they got acquihired.

## the cache

attic runs on a hetzner vps via a small nixos module:

```nix
services.atticd = {
  enable = true;
  environmentFile = config.age.secrets.attic-jwt-secret.path;
  settings = {
    listen = "127.0.0.1:8081";
    database.url = "postgres://atticd@localhost/atticd?host=/run/postgresql";
    storage = {
      type = "local";
      path = "/mnt/storagebox/attic";
    };
    compression.type = "zstd";
    chunking = {
      nar-size-threshold = 65536;
      min-size = 16384;
      avg-size = 65536;
      max-size = 262144;
    };
  };
};
```

the nars are stored on a hetzner storagebox, cifs-mounted into the vps at
`/mnt/storagebox`. storageboxes are very cheap bulk storage, and i already had
one for other storage purposes, so that works well. the vps and the storagebox
are in the same hetzner region, so despite being network storage it's pretty
quick:

```bash
❯ ping -c 5 -q u554215-sub1.your-storagebox.de | tail -1
rtt min/avg/max/mdev = 0.508/0.563/0.615/0.036 ms

❯ dd if=/dev/zero of=/mnt/storagebox/boop bs=1M count=256 conv=fsync
268435456 bytes (268 MB, 256 MiB) copied, 0.664826 s, 404 MB/s

❯ dd if=/mnt/storagebox/boop of=/dev/null bs=1M iflag=direct
268435456 bytes (268 MB, 256 MiB) copied, 0.935912 s, 287 MB/s
```

half a millisecond round trip and a few hundred MB/s in either direction is
plenty for serving nars.

the database i'm using is postgres. attic defaults to sqlite, which worked fine
until multiple CI jobs were pushing to the cache at once, at which point it
couldn't keep up with the concurrent writes and pushes would
[fail with timeouts](https://github.com/samiser/nix-configs/actions/runs/32228526910/job/95993255311#step:4:7770).
the vps was already running postgres for other services, so pointing atticd at
it was easy enough, and the timeouts haven't come back since.

the chunking settings are also worth mentioning. attic splits nars into
content-addressed chunks, so when a package rebuilds with only a small change,
only the changed chunks get stored and uploaded. my nixos system closures change
a little on every flake update, so this saves a lot of space over storing each
closure in full. after months of pushing on every commit for 7 hosts, and with
not particularly aggressive garbage collection, the entire cache is only 5.5GB:

```bash
❯ du -hs /mnt/storagebox/attic/
5.5G    /mnt/storagebox/attic/
```

atticd only listens on localhost, and caddy reverse proxies it at
`cache.samiser.xyz` with tls. the jwt signing secret is stored as an
[agenix](https://github.com/ryantm/agenix) secret, so nothing sensitive is in
the repo.

with the server up, creating the cache and a token for CI is done with the attic
client:

```bash
❯ attic login local https://cache.samiser.xyz [admin-token]
❯ attic cache create main
❯ attic make-token --sub github-actions --validity 1y \
    --pull main --push main
```

that token goes in the repo's github actions secrets as `ATTIC_TOKEN`.

then every host trusts the cache as a substituter via a shared module:

```nix
nix.settings = {
  substituters = [ "https://cache.samiser.xyz/main" ];
  trusted-public-keys = [ "main:xTlqL+c6HRCxNLtRdVu+TElyY+HD9WiXQn0fSetkbFk=" ];
};
```

so any machine rebuilding its config will pull anything CI has already built.

## the CI

the goal of the github actions workflow is for every push, build every host
configuration and devshell in the flake, then push the results to the cache.
that way, by the time i run `nixos-rebuild switch` on an actual machine,
everything is pre-built and it's just downloading rather than building.

### discovering what to build

i didn't want to hardcode the list of hosts in the workflow, so the flake
exposes its own CI matrix. i wrote some nix to collect every nixos
configuration, darwin configuration and devshell into `checks`, grouped by
system, then flatten that into a list of `{ name, system }` pairs exposed as
`ciMatrix`.

the first job in the workflow evaluates that and maps each system to a github
runner:

```yaml
matrix="$(nix eval --json .#ciMatrix | jq -c '
  {"x86_64-linux": "ubuntu-latest",
   "aarch64-linux": "ubuntu-24.04-arm",
   "aarch64-darwin": "macos-latest"} as $os |
  {include: map(. + {os: $os[.system]})}')"
```

so adding a new host to the flake automatically adds it to CI, on the right
architecture, with no workflow changes.

### skipping work that's already done

each matrix job then builds one check. but before building anything, it
evaluates the derivation and asks the cache if the output already exists:

```bash
- name: Evaluate
  run: |
      attr='.#checks."${{ matrix.system }}"."${{ matrix.name }}"'
      eval="$(nix eval --json "$attr" --apply 'd: { drv = d.drvPath; out = d.outPath; }')"
      out="$(jq -r '.out' <<< "$eval")"
      hash="$(basename "$out" | cut -d- -f1)"

      if curl -sf -o /dev/null "$ATTIC_ENDPOINT/$ATTIC_CACHE/$hash.narinfo"; then
        echo "cached=true" >> "$GITHUB_OUTPUT"
      fi
```

the key thing is the http request for the `.narinfo` of the output path. if it's
there, the build and push steps are skipped entirely and the job finishes in a
few minutes (basically just evaluation time). for pushes that only touch one
host, or for re-runs, most of the matrix gets skipped like this.

if the output isn't cached, the job builds it and pushes both the derivation and
its outputs:

```bash
push --no-closure "$ATTIC_CACHE" "$DRV"
push "$ATTIC_CACHE" "${outs[@]}"
```

`push` is a small retry wrapper, because uploading a few gigabytes of nars from
a github runner occasionally fails partway through (failures were much more
common when i was using sqlite, but i kept the retries anyway). the push step is
also `continue-on-error` since a failed cache push shouldn't fail CI, given the
actual build succeeded. it just emits a warning in the job summary so i know the
cache is behind.

### pushing flake inputs too

there's also a separate job that pushes the flake's inputs to the cache:

```bash
paths=()
while IFS= read -r path; do
  paths+=("$path")
done < <(nix flake archive --json | jq -r '[.. | .path? | strings] | unique[]')

attic push "$ATTIC_CACHE" "${paths[@]}"
```

`nix flake archive` gives you the store paths of every input (nixpkgs,
home-manager, etc). caching these means machines and CI runs don't have to
re-fetch input tarballs from github, and evaluation-only operations get faster
too.

### why no DAG ??

you may have realised that because each job builds its host independently, jobs
will sometimes end up building the same derivations at the same time,
duplicating work. if you noticed this, congratulations, you're a discerning
reader!

a smarter approach would be to resolve the dependency graph up front into a DAG,
build each shared derivation once, then build the hosts on top. but scheduling
that across github runners would be pretty complicated, so i thought about it
for a bit then decided against it (for now...)

## closing thoughts

with this setup, flake updates now mean CI does all the building once, and every
machine just downloads the results. this makes deploying updates to my servers
particularly nice, but even just rebuilding my desktop or darwin host is
typically significantly faster.

if you're maintaining a multi-host nix flake and running into the same
_rebuilding everything every time_ annoyance, i'd definitely recommend giving
attic, or just caching for personal uses in general, a go. thanks for reading!
:)
