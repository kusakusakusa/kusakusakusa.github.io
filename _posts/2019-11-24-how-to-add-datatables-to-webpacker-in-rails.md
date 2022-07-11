---
title:  "How To Add Datatables To Webpacker In Rails"
date:   2019-11-24 09:00:00
permalink: how-to-add-datatables-to-webpacker-in-rails
---

This is a documentation of using datatables with the latest version of `Rails` (`6.0` at the time of writing) that uses `webpacker` as the default `Javascript` compiler.

I found some difficulty in looking for documentation of integrating this in the new `Rails` away from the lands of `Sprocket`.

Hopefully this can help you and my future self when I come back to understand what I did to make my codes work instead of leaving it to God.

INSERT MEME Only God Knows

## Datatables and custom styling

`Datatables` ship with its core files and some [default styling packages](https://datatables.net/manual/styling/) with major `CSS` framework like [`Bootstrap`](https://getbootstrap.com/) and [`Foundation`](https://get.foundation/). Taking Bootstrap 4 as the example framework, install the packages below using `yarn`.

```
"datatables.net": "^1.10.19",
"datatables.net-bs4": "^1.10.19",
```

These are the latest versions at the time of writing. They will be added to the `package.json` file.

Require Datatables
In your javascript file (place it in `application.js` for now), require the file and initialize `datatables`.

```js
require("datatables.net")
require('datatables.net-bs4')
require("datatables.net-bs4/css/dataTables.bootstrap4.min.css")

const dataTables = [];

document.addEventListener("turbolinks:load", () => {
  if (dataTables.length === 0 && $('.data-table').length !== 0) {
    $('.data-table').each((_, element) => {
      dataTables.push($(element).DataTable({
        pageLength: 50
      }));
    });
  }
});

document.addEventListener("turbolinks:before-cache", () => {
  while (dataTables.length !== 0) {
    dataTables.pop().destroy();
  }
});
```

Let me explain what each line does.

On line 1, we import the core `datatables` `js` files. These `js` files adds the standard search and sorting functions of datatables as well as wire up any of your custom configurations.

On line 2, the javascript that will work with `Bootstrap` 4 elements. It will add elements to the web page for, for example, the pagination feature using the common `Bootstrap` classes like `row` and `col-*.`

On line 3, it imports the custom `css` file that are required by the `datatables` `JavaScript` function but are not present in default `Bootstrap` stylings. Yes, we are importing the CSS files in a javascript file. Webpack will compile this `JavaScript` file into the public/packs folder and take care of loading the css into the webpage albeit via javascript. Note that if you set the extract_css option as true in the webpacker configuration, it will instruct webpacker to compile the css into a standalone file, instead of loading it as part of the `Javascript` code. Hence, you will need to rely on `stylesheet_pack_tag` to load the `css` file in the page for the styling to work.

Line 5 is where we declare a `datatables` array variable to be accessed within this module that is this script. This is a critical step for `DataTables` to play well with `Rails` in a turbolinks powered environment. The role of this variable is to store all instances of the tables that have been initialized.

The next 2 blocks of code add 2 listeners to the `DOM`.

The first triggers the `dataTable()` function on the desired elements that bear the class data-table. This sets up the pagination, search, sort etc functionalities that make datatables so powerful and simple on your table element. The event this occurs on is `turbolinks:load`, which is when the url changes and the page loads. Each element is initialized and stored in the `dataTables` array variable. The 2nd listener will reference them.

The second listener will destroy each of the `dataTable` instance that are stored in the namesake variable, if any is present. It is triggered during the `turbolinks:before-cache` event, which takes place when the page navigates away. This step is crucial to remove the elements that were added when the datatables script is evaluated, like the search bar and the pagination elements. If this is not done, there will be extra elements appearing on the webpage when the user navigates back through the browser history as mentioned in [this Github issue](https://github.com/turbolinks/turbolinks/issues/106).

NOTE that it is important NOT to name the class of your elements as “dataTable” as they will get destroyed in the process. If that happens, when the user navigates forward and back again or vice versa, the element will not be picked up and the `dataTable()` function will not be executed.

## Optimizing

```slim
= javascript_pack_tag 'custom/datatables', 'data-turbolinks-track': 'reload'
= stylesheet_pack_tag 'custom/datatables'
```

Not every page has a table that you will like to initialize the datatable functionalities on. You should only require this in pages that require the code to be executed. In this way, the initial load time of your page will be reduced by not downloading the extra files that you will not use and affect the page speed of innocent pages.
This means downloading 2 files instead of one which can affect page speed due to having to make 2 request instead of 1. However, the resultant overheads from the http requests are unavailing considering these are javascript files that are not [render-blocking resources](https://web.dev/render-blocking-resources/) and they can be loaded asynchronously to mitigate it.
Put the above code snippet in another js file in the app/javascript/packs. Webpacker will pick up this file as another entry point and compile the `js` asset that you can add separately.
Call this js file in the page that require it as such:

Once again, you will find stylesheet_pack_tag useful only if you have enabled extract_css in the webpacker configuration. It will be responsible to load the compiled (or ‘extracted’ in this context) `css` file.

## Playing Well With Turbolinks

```
# application.html.slim
head
  = yield :javascript_in_head
body
  = yield

# specific/page.html.slim
body
  - content_for :javascript_in_head do
    = javascript_pack_tag 'my-datatables-scripts', 'data-turbolinks-track': 'reload'
```

Be careful of where you load this javascript file. Make sure to load it in the head html element tag because turbolinks will only handle the javascript files loaded in the head and not the body html element tag.
To do this in the page, use the `content_for` helper.

Rails will insert the javascript file in the head section at line 3 the for the given page in the head section of the page’s layout, ensuring that turbolinks perform hooks on the javascript file as well.
