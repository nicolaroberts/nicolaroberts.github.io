---
layout: post
title: "Setting up this blog site: github, jekyll, and the hpstr theme"
tags: [web, git, jekyll]
---

This blog site is powered by [Jekyll](http://jekyllrb.com/) and hosted for free using [GitHub pages](https://pages.github.com/). The design is thanks to [Michael Rose](https://github.com/mmistakes) and his [HPSTR](https://mademistakes.com/work/hpstr-jekyll-theme/) Jekyll theme. 

## Basic steps

* Fork the [HPSTR Jekyll theme repo](https://github.com/mmistakes/hpstr-jekyll-theme)
* Clone locally
* Create a new repo on GitHub called `username.github.io`
* Reset the primary remote repository (origin) of the local cloned copy so that it points to `https://github.com/username/username.github.io.git`
* Delete non-master branches
* Make edits to the source files as desired (see below)
* Push changes to remote origin on GitHub 
* GitHub automatically builds a website from the source in repo `username.github.io`, giving it the address `http://username.github.io/`


## Personalise content


Edit these files to personalise site content:

* `_config.yml`
* `_data/navigation.yml` defines links on side menu
* `_posts/` remove default template posts and start writing new content
* `about/index.md` About page
* `README.md` the README visible on the GitHub source page


## Build the site locally

Build the site locally and check any updates before pushing the changes to the public face on GitHub. 

First time:

* `sudo gem install bundler`
* `sudo gem install jekyll`
* In folder with the website repo: `bundle install`

Every time:

* `bundle exec jekyll build`
* `bundle exec jekyll serve` 
* Paste local server address into browser
* Simply refresh the browser to incorporate edits on the go

Unfortunately, links in the local development version still point to the published online site without the edits. To get the links pointing between local in-development pages, comment out the `url` field in the `_config.yml` file. Don't keep and commit that though!



## Enable DISQUS comments system

* Get Disqus account, click 'Add Disqus To Site', fill out details
* Will create site `https://disqus_shortname.disqus.com/admin/moderate/`
* Edit `disqus_shortname` field in `_config.yml`


## Google analytics and verify

To do. 