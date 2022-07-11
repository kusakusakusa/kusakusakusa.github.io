---
title:  "Handling DOM Elements From link_to remote: true Callback"
date:   2020-03-10 09:00:00
permalink: handling-dom-elements-from-link_to-remote-true-callback
---

This is a documentation of how to handle the response from a link_to remote: true API call and manipulate the DOM with minimal Javascript code.

## Motivation

In the past, I use Javascript to add a click listener on a button element in order to make jquery.ajax() API call to my Rails server.

A typical use case would be to delete a row in a list. While I can make a resource delete request to the Rails server and it will reload the page with the new list the RESTful way, this UX flow does not work out in some cases.

Hence, I had to use the jquery.ajax() way to work things out. I did not use the link_to with remote: true helper because I thought there is no way for me to listen for the response and react.

The the only way I can listen on the callback and handle the DOM element thereafter, without having to reload the page was to use success callback the jquery.ajax(), or so I thought.

## The Magic

So a better way is to use the link_to remote: true helper to render out the element without fuss the Rails way. And the key step is to add a Javascript listener on the element.

```erb
<!-- page.html.erb -->
<div data-model-id="<%= @model.id %>">
  <%= link_to 'DELETE', model_path(@model), method: :delete, class: 'delete-model', data: { confirm: 'Are you sure?' }, remote: true %>
</div>
// page.js
$(document).on('ajax:success', '.delete-model', event => {
  const [response, status, xhr] = event.detail;
  $(`.parent-row[data-model-id="${response.model_id}"]`).remove();
  alert(response.message);
});
```

The listener will listen for a ajax:success callback on the element that made the API call. Upon triggered by the event, it executes the block of code in its callback function. In this callback function, we will receive the data passed from the backend, which we can use to remove the DOM element as required.

Note that you might not want to use $(document).on() in a turbolinks environment as the listener will be added every time the page changes. A particular use case is documented [here](https://www.vic-l.com/integrating-recaptcha-v3-with-turbolinks-in-rails/).

We can add a `ajax:error` listener as well to handle errors.

## The Advantages

INSERT IMAGE javascript crying

This is a no hassle method of writing code in ruby (well, for the rendering of the element at least). The old way that I do, which is to use the jquery.ajax() method, requires more tear-inducing Javascript code to conjure. For a full stack Rails developer, it is not the most welcome.
On top of that, using Rails helper to render out the HTML element allows us to make use of the various Rails helpers to supercharge our development speed.

Url route helpers parse out the actual RESTful route to call with ease. Since it is dynamically interpolated, no code change is required should there be a route change.

We can also still take advantage of rails-ujs which has some handy features commonly needed for development. In the example above, I added a data-confirm attribute. This will be [trigger rails-ujs to ask for confirmation](https://edgeguides.rubyonrails.org/working_with_javascript_in_rails.html#confirmations) before proceeding with the request, and gracefully abort the operation should the user cancel the confirmation.

This will require proper setup with the new Rails 6 version. Check out my article on [how to properly setup Rails 6 with bootstrap](https://vic-l.github.io/setup-bootstrap-in-rails-6-with-webpacker-for-development-and-production), and of couse, integrate the new rails-ujs in its brand new frontend paradigm running on webpack.

## Conclusion

Utilizing rails helpers as much as possible will exude the strength of the Rails framework even more, which is rapid development. This method of listening to remote API calls and act accordingly from the response allows exactly this.
