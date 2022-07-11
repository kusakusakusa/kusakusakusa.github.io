---
title:  "Why Is Render JSON And Return Not Working In My Controllers"
date:   2019-11-06 09:00:00
permalink: why-is-render-json-and-return-not-working-in-my-controllers
---

Recently, I stumbled upon on an unexpected error when i was refactoring my code to follow the style guide of [`rubocop`](https://github.com/rubocop-hq/rubocop), `Ruby On Rails` very own linter.

The [`and`/`or` style guide](https://www.rubydoc.info/gems/rubocop/RuboCop/Cop/Style/AndOr) recommends that the logical operator `&&` be used in place of `and`. When I did that, my controllers started to break my tests.

## Return JSON Did Not In Fact Return

I have a controller action that looks like this. It does a check on the current user and returns early with a custom `json` if the condition is right, instead of continuing the propagation to look for the view file corresponding to itself.

```ruby
def show
  if current_user.one_piece_fan?
    render json: {
      messsage: 'My nakama!'
    } and return
  end
end
```

This works fine, but when I changed the `and` to `&&,` this custom `json` was not returned. What went wrong?

## && vs and

There is a very subtle difference between the two. It is their **precedence order**.

In a line of code consisting of multiple operations, it plays a part in deciding what gets evaluated first. This turns out to be fairly crucial for ruby that has stripped itself of the non-human-friendly characters like brackets and semi-colons. Let’s take a look at the code below.

INSERT IMAGE outputs of && vs and

Did that surprise you?

Well, this is all because of precedence.

In the first line of code, and has a lower precedence than the assignment operator =, hence the assignment took place before the logical `AND` operation is carried out with `false`. In other words, this is actually how it looks like had ruby still have its clothes on.

```ruby
(s = true) && false
```

Hence, the `false` value returned from this line of code is referring to the result of the `&&` operation. And when one of the operand is `false`, the result will be `false`.

INSERT IMAGE multiply by 0

As for the third line of code, it works just like how most people would commonly interpret it. The result of the && operation is assigned to the s variable, and it subsequent false value is the value that the variable s now holds.

## Returning Early In Controllers

So back to the case of controllers.

```ruby
class ApplicationController < ActionController::API
  def render_and
    render json: {
      message: 'Using "and return"'
    } and return
  end

  def render_amp
    render json: {
      message: 'Using "&& return"'
    } && return
  end

  def render_amp_with_brackets
    render(json: {
      message: 'Using "&& return" with brackets'
    }) && return
  end
end
```

Base on the snippet above, the actions `render_and` and `render_amp_with_brackets` will work just like how you would have expected. They will return the render function early and stop the controller from propagating further.

As for the `render_amp` method, it is rendering the result of the `&&` operation between the return function and the hash. Essentially, it looks like this.

```ruby
render({ ... } && return)
```

Since there is no eventual return in this render function, the controller will further propagate and carry out its search for the view corresponding the the action.

## Final Thoughts

I hope this has help us understand our and and &&s better!

Credits to [this stackoverflow answer](https://stackoverflow.com/questions/39629976/ruby-return-vs-and-return/39630299#39630299).

