---
tags: [blog, web, python]
title: building a javascript-free single-page static site compiler
summary: "How I created the website you're reading this post on!"
date: 2022-03-16
publish: no
---
Building my own website has always been something of interest to me.
I really like the idea of having my own space on the internet, and it's been something I've attempted to do multiple times.

My first efforts were several iterations of static websites, me learning HTML and CSS by tinkering about and not really getting anywhere.
Sadly these don't exist anywhere online as I never actually knew how to put them on the internet, but just trust me when I say they were poorly designed and unmaintainable messes of HTML files all over the place.

After getting a bit more comfortable with programming and learning a bit more about web technologies, I decided to tackle building my own CMS system with Django.
That was my personal website for a long time, and it served me well.
It was particularly good as a CV item, as I implemented the full stack of the site.
It was running on my homelab, in a container I provisioned, by a systemd service I wrote.
Pretty cool!

You can see what that site looked like with the [wayback machine](https://web.archive.org/web/20191219213122/https://www.samiser.xyz/).

The source code can be found [here](https://github.com/Samiser/my-blog), and if you took the time to read through it (I wouldn't recommend it) you would notice that it's not great.
It ended up becoming a pain to maintain and I kind of gave up on try to improve it because it was so overcomplicated.

I decided this year to create some new website infrastructure from scratch to address this.
Fortunately I had the foresight to write my blog posts in markdown, so it would be trivial to port them to a new system.
I just had to figure out what that system would look like.

https://css-tricks.com/a-whole-website-in-a-single-html-file/
- saw a few "single page" websites
- the idea intrigued me
- decided to build a site compiler for a single page site
- this site is pretty simplistic
- i like it that way

# Single page site
Before going into the compiler details, I wanted to cover the design and philosophy behind the site.
The three defining attributes I'm going over here are co-dependant in some ways, but I still think it's useful to touch on them individually.

## javascript-free
The site doesn't serve or execute any javascript.
I'm not opposed to javascript, just think it's not necessary on a simple site like this.
Mostly doing it this way for fun.

## single-page
There's only one "page" that is loaded, content is rendered only when needed.
This is a typical design pattern for modern web apps and is a nice user experience.

## static site
Webserver is just serving pre-made HTML and CSS.
This is the feature with the most benefits:
- Webservers are well optimised to serve static content
- Content being generated on the server-side is more flexible
- It's trivial to scale up the serving of a static site
- Far more secure than dynamic web apps

# How it works
This website uses a very simple trick to render content dynamically: the CSS `:target` pseudo-class.

CSS pseudo-classes are a very useful way to define behaviour by the *state* of an element.
The most common one i've seen is `:hover`, which allows you to style an element when the mouse hovers over it.

`:target` selects the element present in the URL after the `#`.
Using CSS, we can show the "selected" page, and hide the rest:

	:::css
	section { /* Hide all sections */
	  display: none;
	}
	
	section:target { /* Show target section */
	  display: block;
	}

You'll notice as you navigate this site the pages in the URL are prefixed with `#`, and this is why!
Each "page" is just a `div` with a unique `#id`, and as you follow each link on this site and the URL is updated, those `div`s become selected via `:target`.
The CSS then shows the selected page.

However, with only that CSS in place, nothing would actually show on loading the website initially because nothing is selected.
To fix this, one of the pages can be made to show despite what is selected:

	:::css
	section#home { /* Show #home by default */
	  display: block;
	}

But now we've introduced a new problem: rendering in a new page will just show them both at the same time, one on top of the other.
This can be resolved by setting the position of all sections to `absolute`, the position to `top: 0`, and to make sure the width and height will cover up the section below:

	:::css
	section {
	  display: none;
	  position: absolute;
	  top: 0;
	  min-height: 100vh;
	  width: 100%;
	}

And that should be it!
Here's the full CSS snippet:

	:::css
	section {
	  padding: calc(6em + 5vw) 5vw 8vw 5vw;
	  background: var(--bgcolor);
	  /* ! Everything below here is needed ! */
	  display: none;
	  position: absolute;
	  top: 0;
	  min-height: 100vh;
	  width: 100%;
	}
	
	section:target { /* Show section */
	  display: block;
	}
	
	section#home { /* Show #home by default */
	  display: block;
	}

With how the site works out of the way, now can discuss how I built the compiler.
# Compiler
The compiler has three stages:

## Parse markdown files
- parse function:
	- for a dir
		- for every .md file in the dir
			- convert to html
			- grab frontmatter metadata
		- return dict of parsed html & metadata

The first stage is to parse the markdown files that represent my pages and blog posts.

I use a single function for this

	:::python
	def parse(dir_string, exclude=[], sort_output=False):
	    out = {}
	    
	    in_dir = os.fsencode(dir_string)
	    
	    for file in os.listdir(in_dir):
	        name = os.fsdecode(file)
	        if name.endswith(".md") and name not in exclude:
	            infile = open(os.path.join(dir_string, name), mode="r", encoding="utf-8")
	            raw_md = frontmatter.loads(infile.read())
	    
	            if "publish" in raw_md and not raw_md["publish"]:
	                continue
	    
	            md = markdown.Markdown(extensions=["codehilite"])
	    
	            out[name[:-3]] = {
	                "content": md.convert(raw_md.content).replace(
	                    "<img alt", '<img loading="lazy" alt'
	                ),
	                "meta": raw_md.metadata,
	            }
	    
	            out[name[:-3]]["meta"]["time_to_read"] = round(
	                len(raw_md.content.strip().split(" ")) / 300
	            )
	    
	            print(out[name[:-3]]["meta"])
	    
	    if sort_output:
	        return OrderedDict(
	            sorted(out.items(), key=lambda x: x[1]["meta"]["date"], reverse=True)
	        )
	    else:
	        return out

## Generate the pages
- template:
	- for each page:
		- page link in navbar
		- page content in it's own section
	- for each blog post:
		- link to post in blog index page
		- blog content in it's own section

## Render pages in template
- main:
	- parse blog posts
	- parse site pages
	- render template with parsed pages and blog posts
	- write output to outdir