---
title:  "How To Use HTML Validation On Flatpickr"
date:   2020-04-29 09:00:00
permalink: natural-sorting-with-datatables
---

The most significant difference between natural sorting and your typical sorting is the order of a string when there are digits involved. For instance, the string “something2” will be placed before the string “something10” under normal sorting, but 10 is more than 2 for that matter.

This default sorting algorithm causes sorting problems with Datatables and we will see 2 ways to solve it, using a natural sort plugin and the data-sort or data-order attribute.

Natural Sort Plugin In Datatables
The documentation for this plugin lives [here](https://datatables.net/plug-ins/sorting/natural). This is how it is implemented in my Rails projects that are running on [Turbolinks](https://github.com/turbolinks/turbolinks).

First, install the datatables plugin package via the command below:

yarn add datatables.net-plugins
In the javascript file, the snippet looks like that.

```js
// app/packs/any.js
require('datatables.net-plugins/sorting/natural’)

document.addEventListener("turbolinks:load", () => {
  if (dataTables.length === 0 && $('.data-table').length !== 0) {
    $('.data-table').each((_, element) => {
      dataTables.push($(element).DataTable({
        pageLength: 50,
        columnDefs: [
          { type: 'natural-nohtml', targets: '_all' }
        ]
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

There is quite a bit of complexity in implementing this. The explanation can be [found in this article](https://vic-l.github.io/how-to-add-datatables-to-webpacker-in-rails) and is largely due to the need to adapt to a turbolinks driven environment.

The key implementation takes place in line 9 and 10. The natural-nohtml type is specified to strip any html during sorting, while the _all value for the target key means to apply natural sort as the default sorting for all columns. More information and configuration options can be found in its [documentation](https://datatables.net/plug-ins/sorting/natural).

## data-sort Or data-order Attribute

We can use the data-sort or data-order attribute of the table cell to indicate to Datatables to use these value to do the sorting instead of the values in the cell.

```html
<td data-order="02">2</td>
<td data-order="10">10</td>
```

This will place the string in the correct numerical order. However, you would have to do the heavy lifting of padding the digits with the appropriate number of 0s.

It is a more useful feature if you want to sort values with a vastly different display value that its actual value that can mess up the sorting order, like date.

```html
<td data-order="1332979200">Thu 29th Mar 12</td>
<td data-order="1354406400">Sun 2nd Dec 12</td>
```

Under normal sorting, the second <td> element will be placed above the first because ‘S’ comes before ‘T’. However, when we dictate the sort order to be using their epoch timestamp, the story will be different.

