---
title: "Come ho fatto questo blog"
date: 2026-02-08 00:00:00 +0000
categories: [tutorials]
tags: [tutorial, blog]
---

# Hostare un blog su github pages gratuitamente

ho semplicemente seguito questo tutorial:

{% include embed/youtube.html id='m1RYsmOMPLs' %}

Consulta la [guida completa originale](https://chirpy.cotes.page/posts/write-a-new-post/) per la formattazione di post.

## Altre informazioni utili

```
puoi usare gli apici per evidenziare ``` blocchi di codice ```
```


> puoi creare una cartella dedicata per ogni post in `/assets/img/posts/first-post`{: .filepath } e aggiungi le tue immagini che potrai richiamare in qualsiasi momento con `![Desktop View](/assets/img/posts/first-post/dog.jpg){: width="300"}_Ecco una immagine di esempio_`.
{: .prompt-tip }



![Desktop View](/assets/img/posts/first-post/dog.jpg){: width="300"}
_Ecco una immagine di esempio_

### Run con watcher

Assicurarsi di aver aggiunto questo file in `/_plugins/watcher-patch.rb`

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