---
title:  "How To Use HTML Validation On Flatpickr"
date:   2020-04-29 09:00:00
permalink: how-to-ssh-into-private-servers-via-a-bastion-without-copying-ssh-keys
---

My go to date picker JavaScript library is [flatpickr](https://flatpickr.js.org/). It has a decent UI that can be customized easily, is lightweight, and is straight forward to implement. There is one particular flaw that I seek to address in this article, that is integrating it with the basic HTML validation.

## Motivation

I want to be able to use basic HTML validation on my date inputs. A date input with a simple required attribute should trigger the HTML validation if it is empty upon form submission.

I also would like to prevent users from being able to enter free text in the text field which is meant for date input.

However, flatpickr comes with a flaw that would demand a hack to make it work the way I want.

## The Read Only Problem

The cause of its flaw is that flatpickr disables itself by slapping the input with a readonly attribute. A readonly attribute will cause the input to elude the grasp of the basic HTML validation during form submission. This is so as HTML validation ignores readonly, disabled and hidden elements.

## The Catch-22 Solution

The immediate solution is to initialize the flatpickr instance with the option allowInput set to true. However, this will allow users to be able to enter free text in the date input, which is what I would also like to guard against.

## The Hack

Hence, my solution here is to implement a listener that will toggle the input’s readonly attribute when the date picker dropdown is active, and toggle it back when the user has moved on from date picking.

## The Code

```js
flatpickr("[data-behavior='flatpickr']", {
  altInput: true,
  altFormat: 'F j, Y',
  dateFormat: 'Y-m-d',
  allowInput: true,
  onOpen: function(selectedDates, dateStr, instance) {
    $(instance.altInput).prop('readonly', true);
  },
  onClose: function(selectedDates, dateStr, instance) {
    $(instance.altInput).prop('readonly', false);
    $(instance.altInput).blur();
  }
});
```
Line 1 initializes the flatpickr instance on elements specified by the selector.

Line 2 to 4 are my custom configuration options. I am placing it here to demonstrate the hack for a non default scenario.

Line 2 indicates my intention to display my date values in an alternative format.

Line 3 indicates the alternative format to display in.

Line 4 indicates the date format that will eventually submitted to the form’s action endpoint.

These configurations will tell flatpickr to create 2 inputs.

One is hidden and meant to be submitted to the backend. This input will inherit the attributes of the original input element that would matter during submission, like the name and id. Let’s call this the hidden input.

The other input is meant for display purpose. The value in it will be shown the desired alternative input format. And since it is display, it will inherit the relevant attributes from the original input element that matters for display. For example, the style, class and in particular, the readonly and required attributes. Let’s call this the display input.

Line 5 removes the default readonly attribute that flatpickr will place on its target input elements.

Line 7 defines a function that will be triggered when the dropdown is active. It will find the display input and set it to readonly. This will prevent users from being able to entering any text that will mess up the input value.

Line 10 undo the effect of line 7 in the likely scenario when the user has finished picking the date and the date dropdown closes.

Line 11 handles the behavior where the input cursor remains on the date input when the datepicker dropdown is deactivated. A possible scenario that this might happen is when the user presses the escape key when the dropdown is active. When that happens, the readonly attribute is gone and the user is able to enter any text on the input. Thus, the blur() function prevents this.
