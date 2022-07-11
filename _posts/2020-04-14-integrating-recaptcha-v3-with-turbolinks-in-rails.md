---
title:  "Integrating reCaptcha V3 With Turbolinks In Rails"
date:   2020-04-14 09:00:00
permalink: integrating-recaptcha-v3-with-turbolinks-in-rails
---

Google has published the latest new version of [reCaptcha V3](https://developers.google.com/recaptcha/docs/v3) and I had to integrate it into my recent Rails projects. The greatest difference between the old version is its improvement in user experience.

It removes the user friction where users are required to click on the notorious “I am not a robot” check box and at times take some spontaneous image verification quiz.

In its place, the new reCaptcha observes the user’s actions on the website to determine if he/she is a genuine human user.

It generates a score which the backend of the website will need to verify against to decide if the score is above the threshold of what is considered a real user.

On the frontend, there’s no more extra step required to submit the form. Pretty neat!

In the midst of integrating it to my project, I had some problems, as usual, with [turbolinks](https://github.com/turbolinks/turbolinks). The biggest of them is navigating between pages. Hence, this article seeks to document the process.

## Initializing

Due to the use of turbolinks, the initialization process is different from what was documented. In fact, there is little to no documentation on the alternative way to initialize the recaptcha library. With reference to [this blog](https://blog.bmonkeys.net/2019/using-recaptcha-v3-with-turbolinks), the initialization step is as such.

Note that I am using the [slim template engine](http://slim-lang.com/) to generate my HTML views.

```slim
// in the < head >
script src='https://www.google.com/recaptcha/api.js?render=explicit&onload=renderCaptcha'
= javascript_pack_tag 'recaptcha', 'data-turbolinks-track': 'reload'
```

I insert this snippet at the head of the pages that requires reCaptcha using the [content_for helper](https://api.rubyonrails.org/classes/ActionView/Helpers/CaptureHelper.html#method-i-content_for).

This method of requiring the file allow us to use a custom function to initialize the grecaptcha object. This thus provide us control as to when we want to initialize the object so as to prevent reinitialization when navigating between pages in a turbolinks environment.

This method is documented in an obscure area in the [recaptcha V3 docs](https://developers.google.com/recaptcha/docs/faq#how-can-i-customize-recaptcha-v3) and is also usable in V2 as documented [here](https://developers.google.com/recaptcha/docs/invisible#js_api).

The javascript function renderCaptcha will be called when the file has loaded, and it is constructed in the recaptcha.js.erb file.

Note that this file is given the attribute data-turbolinks-track with a value of reload. This implies that when we navigate between pages where the [tracked assets required are different, the site will do a full reload instead of going through turbolinks](https://github.com/turbolinks/turbolinks-classic#asset-change-detection). In particular for this case when navigating from a page with recaptcha to another without recaptcha, there will be a full reload of the page as the tracked asset, recaptcha.js.erb is no longer present.

This ensures that the recaptcha library is downloaded again and the renderCaptcha function is called when the script is loaded for initialization.

Let’s take a look at the content of the renderCaptcha function.

## The Javascript

```js
// recaptcha.js.erb
window.renderCaptcha = function() {
  document.grecaptchaClientId = grecaptcha.render('recaptcha_badge', {
    sitekey: "<%= Rails.application.credentials.dig(Rails.env.to_sym, :recaptcha, :site_key) %>",
    badge: 'inline', // must be inline
    size: 'invisible' // must be invisible
  });
  window.pollCaptchaToken();
}
window.pollCaptchaToken = function() {
  getCaptchaToken();
  setTimeout(window.pollCaptchaToken, 90000);
}
window.getCaptchaToken = function() {
  grecaptcha.execute(document.grecaptchaClientId).then(function(token) {
    document.getElementById('recaptcha_token').value = token;
  });
}
document.addEventListener("turbolinks:load", () => {
  $('#contact-form').on('ajax:success', event => {
    ...
    $('#contact-form').trigger('reset');
    window.getCaptchaToken();
  });
});
```

Firstly, note that this is an erb file. This allows us to render ruby variables into javascript and compiled by Webpacker during build time. Refer to this documentation on [installing erb with Webpacker](https://github.com/rails/webpacker/blob/master/docs/integrations.md#erb) or my article on [setting up bootstrap with Rails 6 and Webpacker](https://vic-l.github.io/setup-bootstrap-in-rails-6-with-webpacker-for-development-and-production) on how to set this up. In this case, I am storing my recaptcha site key using the [new Rails way since 5.2](https://blog.engineyard.com/rails-encrypted-credentials-on-rails-5.2) and parsing it in the javascript file during build time for consumption.

The `renderCaptcha()` initializes the recaptcha script and renders the recaptcha badge on an HTML element with the id recaptcha_badge. Once initialized, the getCaptchaToken() will then retrieve the recaptcha token and utilize it in its callback function. I will be setting the value of the an input element with the id recaptcha_token. This input will be sent along to the backend for the backend to use for verification. More on the views in a bit.

My logic is to poll the new token every 1.5 minutes as the token expires every 2 minutes. The 30 seconds buffer should be sufficient for my backend, which will receive the recaptcha token, to verify with the recaptcha server before the token expires. I have split up pollCaptchaToken() with the actual function getCaptchaToken() to get the token because I will be using getCaptchaToken() explicitly after I submit the form to refresh the token.

Note the use of window and document here. These objects persist in between page navigations in a turbolinks environment. Hence, they provide us a way to keep track of data so we do not initialize the function multiple times while navigating back and forth. And the key data to track here is the grecaptchaClientId on the document object. It tracks whether we have initialized the recaptcha script already or not.

That said, remember the data-turbolinks-track attribute with the value reload added to the script? Once again, it ensures the page fully reloads should the tracked assets be any different in between page navigations. This ensures 2 things:

- Prevents multiple initilizations occurring while navigating between pages because grecaptchaClientId is not null
- Ensures initialization will occur when traversing from a page without the recaptcha script due to a full reload. Otherwise, we will have to wait for the polling function to happened before we can get our token, and that will be disastrous should the user submit the form with a blank token before that.

Lastly, I add an `ajax:success` event listener on the form to handle a successful remote javascript call to my Rails backend. Note that I cannot add the listener on the document object as such:

```js
$(document).on('#contact-form', 'ajax:success', function() { ... })
```

As the document object persist between navigation, it will result in the event listener being added each time a page navigation occurs, hence causing undesirable effects.

The View

```slim
#recaptcha_badge.d-none data-turbolinks-permanent=''
= hidden_field_tag :recaptcha_token, '', data: { turbolinks_permanent:'' }
```

The `#recaptcha_badge` object will hold the badge of the reCaptcha. You can add styling in whatever way you want, but I am using the bootstrap d-none css class to hide it totally as I do not need it.

The `hidden_field_tag` renders a hidden input field where I will store the recaptcha token.

These elements are given the data-turbolinks-permanent attribute. This is a crucial step. It ensures that the elements with the same id are not re-rendered in between page navigations in a turbolinks environment. [Persisting the form element across page loads](https://github.com/turbolinks/turbolinks#persisting-elements-across-page-loads) prevents the input from losing the recaptcha token. Without it, we will need to wait for the polling function to occur again after navigation before we are able to get a new recaptcha token for submission.

That said, the data-turbolinks-permanent on the #recaptcha_badge may not be necessary. But I am just persisting it across pages as well for trivial reasons.

Of course, make sure the input is within the form element so that it gets passed to the backend upon submission.

## Conclusion

This new recaptcha user experience is a definitely a good step towards improving conversion. But integrating with turbolinks is troublesome as always. I hope this article helped to address it, and provide enough explanation on why each step are required adequately.
