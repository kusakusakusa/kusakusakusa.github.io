---
title:  "Critical CSS in Rails 6"
date:   2022-05-11 09:00:00
permalink: critical-css-in-rails-6
---

# Critical CSS in Rails 6

The idea here is to load the css that is precompiled by webpacker along with the html of the page so that the page can go on to load itself in a presentable appearance without having to wait for the required css to be downloaded.

We are going to read the compiled css file and load it inline it in the head element of the webpage.

始めましょう！

## Preparing The Development Environment

Firstly, you need to set extract_css as true in your webpacker.yml configuration file for your development environment.

```yaml
default: &default
  ...
  development:
    <<: *default
    extract_css: true
    ...
```

This will ensure that the file is also compiled in the development environment, and can be found in the public/packs folder.

Note that the file is still kept in memory and not physically stored in that folder. I believe this is due to some webpack-dev-server magic happening.

In production, the file will be compiled and physically stored in that public/packs folder.

## Picking The Critical css Files To Compile

Next, you need to add the relevant css files in a javascript file in the app/javascript/packs folder.

This folder app/javascript/packs is where Webpacker will look at, by default, to get the files it needs to compile.

And if these file contains css codes, its css related loaders will compile them accordingly into a css with the same name as the javascript file itself. It will then be placed in the public/packs folder as mentioned.

In this snippet below, we are compiling only some of the modules of the bootstrap library into a javascript file called common.js. We should expect a file named common-somegibberish.css to be compiled.

```js
// app/javascript/packs/common.js
require("bootstrap/scss/bootstrap-grid.scss")
require("bootstrap/scss/bootstrap-utilities.scss")
```

At this point of time, this resultant compiled css file is a legit css file that can be added to any html file and read by any browser, or maybe non Internet Explorer browser. You can proceed to read off this file into your application.html.erb file if you had [removed the fingerprints on the compiled css file](https://vic-l.github.io/how-to-remove-fingerprinting-in-assets-with-rails-webpacker).

If not, this is how you can dynamically retrieve the file.

## Lookup Compiled CSS File

A simple `File.read` to read off the content of the file, complemented by the Webpacker.manifest.lookup method to search for the file with the correct fingerprint.

```slim
<!-- application.slim -->
doctype html
 html lang="en"
   head
     ...
     style
       = File.read(File.join(Rails.root, 'public', Webpacker.manifest.lookup('common.css'))).html_safe
       ...
```
