---
lang: en
hidden: true
lang_ref: hello-world
permalink: /en/posts/Hello-world/
title: "How I made this blog"
date: 2026-02-08 00:00:00 +0000
categories: [tutorials]
tags: [tutorial, blog]
image:
  path: /assets/img/posts/first-post/cover.jpg
  alt: A Jekyll blog with Chirpy theme hosted on GitHub Pages
---

# Host a blog on github pages for free

I simply followed this tutorial:

{% include embed/youtube.html id='m1RYsmOMPLs' %}

Check out the [original complete guide](https://chirpy.cotes.page/posts/write-a-new-post/) for post formatting.

## Other useful information

```
puoi usare gli apici per evidenziare ``` blocchi di codice ```
```


> you can create a dedicated folder for each post in `/assets/img/posts/first-post`{: .filepath } and add your images that you can recall at any time with `![Desktop View](/assets/img/posts/first-post/dog.jpg){: width="300"}_Here is an example image_`.
{: .prompt-tip }


![Desktop View](/assets/img/posts/first-post/dog.jpg){: width="300"}
_Here is an example image_

### Run with watcher

Make sure you have added this file in `/_plugins/watcher-patch.rb`

```
# frozen_string_literal: true
require 'jekyll-watch'

module Jekyll
  module Watcher
    extend self

    alias_method :original_listen_ignore_paths, :listen_ignore_paths

    def listen_ignore_paths(options)
        original_listen_ignore_paths(options) + [%r!.*\.TMP!i]
    end
  end
end
```
run:

```
bundle exec jekyll s
```
