---
title:  "Rails select helper with default selected option disabled and prompt"
date:   2020-03-12 09:00:00
permalink: rails-select-helper-with-default-selected-option-disabled-and-prompt
---

This is a documentation once and for all on how to render a select tag in rails view with a disabled option that is preselected.

## Motivation

I find myself having to refer to stackoverflow often for the solution because it just isn’t intuitive. It requires some sort hacking before it can be rendered the way I needed it. Often, I would not understand the purpose of writing the code that way unless I can remember the purpose and  see the end goal of what it is trying to achieve.

Code that is not self-explanatory is a sign of code smell. That is so not Rails.

## The Old Way Of Coding

```erb
<= tweet_form.select :user_id,
	options_for_select(
		@users.map { |user| [user.name, user.id] },
      selected: tweet_form.object.user_id || 'Please select user',
      disabled: 'Please select user'
    ),
	{},
	{ class: 'form-control' }
%>
```

The way I used to code out the list of options is to create an array that contains yet another array as shown above. In the inner array, the first value will be the label that users will see when selecting the option from the list, while the second value is the actual value to be passed to the backend.

Here is the part that raises question. In line 3, I prepend the array of users using the + operator with an array containing Please select user. This is meant to be the first value to be selected so that it can act as a prompt for the select tag.

I use the options_for_select helper to make it a little easier to set the with the selected and disabled options. The selected key will select the prepended array’s value as the default value for the select tag, and the disabled key will set it as disabled so that users cannot choose it and send an invalid value to the backend.

Hence, without knowing the end goal, the code from line 3 to 5 will raise eyebrows. There has to be a better way and indeed there is. But first, let me touch on the remaining lines for completion sake.

Line 7 is for other options for the [Rails select helper](https://api.rubyonrails.org/v6.0.2/classes/ActionView/Helpers/FormOptionsHelper.html#method-i-select), like include_blank which we are not using.

Line 8 is for additional HTML attributes.

## The New Way Of Coding

Rails 6 has added the [prompt helper](https://github.com/rails/rails/pull/32087) in the Rails select tag to achieve exactly this purpose. Let’s compare the new way of writing that snippet.

```erb
<= tweet_form.select :user_id,
	@users.map { |user| [user.name, user.id] },
	{
      selected: tweet_form.object.user_id || "",
      disabled: "",
      prompt: 'Please select user'
    },
	{ class: 'form-control' }
%>
```

We are no longer using the options_for_select helper as we do not need its selected and disabled options for any more unsettling hackery.

In its place, we use the prompt key in the line 6 under the options argument for the Rails select helper to give achieve the same result.

The disabled key here makes the prompt an unselectable option. Leave it out if your UX allow selection of nil value. You may need to engage the include_blank key here as well.

The selected key sets the selected value conditionally to be the prompt’s unless the form object already has a user_id value. This part is a little quirky; the concept of selecting the value of the form object should already be implemented by default without having to write out the conditional code as in line 4. I believe this is constructed to fit all scenarios for all UX requirements, but I can’t really be sure. I’ll check it out someday.

Note that by using prompt, the prompt will not be present as one of the options if the form object already has a value.

## Conclusion

This is a lot neater and much more like the Rails we know. There is no more questionable code and every line and option has a clear purpose.

