---
title:  "Setup Bootstrap In Rails 6 With Webpacker For Development And Production"
date:   2019-12-09 09:00:00
permalink: setup-bootstrap-in-rails-6-with-webpacker-for-development-and-production
---

This is a documentation on how to setup Bootstrap 4 in Rails 6 using Webpacker. As the framework shifts away from sprockets and the asset pipeline to embrace the dominating methodology of handling frontend affairs in the Javascript world that is [webpack](https://webpack.js.org/), we have to adapt along.
The way to setup a css framework to bootstrap your application has undergone a revamp, and this article seeks to cover the essential steps to set it up.

## Pre-requisites

This article will assume you have set up all the required tools required for a typical Rails 6 application.

The main extra tool you will need as compared to previous versions of Rails is the [`yarn`](https://yarnpkg.com/) package manager. You can install yarn on your computer via various ways based on your preference and your OS.

We will not be covering it in this article.

## Setting Up Bootstrap

With the shift in paradigm of handling front end assets, we no longer install front end libraries using gems. In the past, these gems are merely wrappers around the Javascript libraries and files which present a number of problems.

First, the latest changes in the Javascript world will take some time to propagate into the Rails realm.

Second, having an intermediate wrapper increase the potential points of failure during the wrapping process.

Third, we are really dependent on the angels who are working on these wrappers. If they do not update the gems frequently, we are stuck with the old features. This can be frustrating if you are waiting for a certain bug fix or a new feature that is already available in the latest release.

To install bootstrap, run this command.

```bash
yarn add bootstrap jquery popper.js

# for bootstrap 5.0
yarn add bootstrap jquery @popperjs/core
```

This command will automatically install the latest bootstrap package in the yarn registries and add its dependency entry and version in your package.json file. Jquery and popper.js are [libraries that bootstrap depends on](https://getbootstrap.com/docs/4.0/getting-started/introduction/), especially in their Javascript department.

## The JS And CSS Files

The main Javascript file, application.js should now reside in the app/javascript/packs folder. This is because Webpacker will now look for all the javascript files in this directory to compile. This is the default setting for Webpacker.

Of course, you can go ahead and change the configuration to your liking. However, keep in mind that Rails promotes convention over configuration. This implies that as much as possible, methodologies and practices should follow a certain default unless absolutely necessary. this has multiple advantages. My favorite one is the portability of code among fellow Rails developers. Developers can easily understand the flow of logic and where to find bugs because they are where are expected to be. This cuts down the development time and cost greatly.

The application.js file should look like this:

```js
require("@rails/ujs").start()
require("turbolinks").start()
require("@rails/activestorage").start()
require("channels")
require("bootstrap")

// stylesheets
require("../stylesheets/main.scss")
```

Line 1 to 4 are the default files already present in the file.

Line 5 adds the Bootstrap Javascript library.

Line 8 adds your custom stylesheet. Now, this file can be placed anywhere. In the above example, the path is relative to where the application.js file is. Hence, the file is placed in app/javascript/stylesheets/main.scss in this example.

Next, we import the Bootstrap stylesheet files in the main stylesheet file.

```scss
@import "bootstrap/scss/bootstrap";
```

Note that we are importing files from the node_modules folder, and not a bootstrap folder placed in the relative path of the current directory of the main stylesheet file.

Also, you do not need the ~ in front of the path to signify that it is from the node_modules folder like you would usually do for other non-Rails project using webpack. The tilde alias in webpack is a default webpack configuration that will resolve to the node_modules folder. While it will still work here, it is not required as the node_modules folder is already configured as part of the search paths that webpack will look for when resolving the modules.

Now, you may be wondering how to the Bootstrap libraries will work without importing any of its dependencies, that are popper.js and Jquery. We will come to that in a minute. Before that, let’s look at the views.

## The Views

Now, we will need to add the javascript and stylesheets files into the page. Following convention in this example, we will add to the application.html.erb layout so that the Bootstrap framework can be accessed in all pages. These lines of code are added in the head section of the layout template.

```erb
<%= stylesheet_pack_tag 'application' %>
<%= javascript_pack_tag 'application', 'data-turbolinks-track': 'reload' %>
```

There are a  number of things that are different from the old implementation.

Line 1 adds the compiled stylesheets path that webpacker will compile. Note that this only happens if the extract_css option is set to true in the webpacker.yml file. More about this later.

As you can see, there is no more stylesheet_include_tag. In the past, this helper method will get files from the public/assets folder, into which the asset pipeline will compile stylesheets and javascript files with other added pre and post processing. Now, everything is going to be done by Webpack.

Here what’s happening.

Webpack will look at application.js and find the stylesheet files that are included in it. Then, using a combination of [Webpack loaders](https://webpack.js.org/loaders/), Webpack will know how to compile and translate the scss syntax, the url paths of assets used etc. into a css file that the browser can read and implement its styling.

These Webpack loaders are already included by the Webpacker and its configurations set up. However, there are many loaders out there that are not included by default. They tend to be less used conventionally and will require manual intervention from your side.

One example is using ruby code inside your javascript files. This requires the rails-erb-loader that will “teach” Webpack to understand the erb syntax. The implementation involves a number of steps, one of which is to append this loader to the Webpack environment.js configuration file. Thankfully, for this case, the community has deemed it a pretty common use case that there is, at least, a [rake task](https://github.com/rails/webpacker#erb) that comes together with the Webpacker gem to set this up easily.

The compilation process mentioned above, however, is not applied in the development environment by default. This is due to the extract_css settings in the webpacker.yml page. More about this and its implications in a bit.

Note that stylesheet_include_tag still works for assets you place in the app/assets folder. However, while that is true, as Rails moves away from the old Sprockets and assets pipeline convention, this is expected to become deprecated in the future.

The Webpacker Configuration File
Lastly, we need to add the dependencies of bootstrap. This takes place in the config/webpack/environment.js file.

```js
const { environment } = require('@rails/webpacker')
const webpack = require('webpack')

environment.plugins.append('Provide', new webpack.ProvidePlugin({
  $: 'jquery',
  jQuery: 'jquery',
  // uncomment below for bootstrap 4.x
  // Popper: ['popper.js', 'default']
  // uncomment below for bootstrap 5
  Popper: ['@popperjs/core', 'default']
}))

module.exports = environment
```

As you can see, we are utilising the [ProvidePlugin function of Webpack](https://webpack.js.org/plugins/provide-plugin/) to add the dependency libraries in all the javascript packs instead of having to import them everywhere.

This is just an example of how we can import files with Webpack in Rails. And in this case, especially for jQuery, it makes a lot of sense as there is a high chance that we will be using it in other javascript files.

Coincidentally, this is how jQuery and popperjs, which are dependencies of the bootstrap library, are made available for the bootstrap library to use them.

## The extract_css Option

There is one last point I would like to touch on. That is the extract_css option in the config/webpacker.yml file.

When set to true, webpack will compile the stylesheet files that were imported into the javascript files into external standalone stylesheets. These compiled files will then be added into the views via the stylesheet_pack_tag helper method as mentioned earlier.

In comparison, when set to false, the stylesheets are not compiled into standalone files. Instead, they are added into the view as a blob during runtime by the the relevant javascript file. This takes place only after the javascript file has been completely downloaded by the browser.

In development mode, the conventional setting for the extract_css option is false, and this has quite a significant implication on how the website will behave.

One, there might be a [flash of unstyled content (FOUC)](https://en.wikipedia.org/wiki/Flash_of_unstyled_content) when the page loads because the javascript files are loaded asynchronously. This is unlike the css files which are blocking resources that will pause the rendering of the website until the file has been downloaded. This asynchronous loading of files allows the website to continue rendering while it waits for itself to be completely downloaded before computing the css blob and insert it into the html source code. If the web page loads before this occurs, the style for the web page is not present, and FOUC will thus occur.

Two, the `stylesheet_pack_tag` is not needed in the development environment using the default setting. Things will seem to work fine only until it is pushed into the production environment where the extract_css option is set to true, desirably and by default.

So make sure to add the stylesheet_pack_tag helper, but only if your javascript is going to compile a stylesheet and your page is reliant on it. If not, you are in for a surprise when it gets pushed to production.

## Conclusion

At this point of time, the application should be running with Bootstrap in place. Do test out how it will differ in the production environment as compared to development.

