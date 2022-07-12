---
title:  "Customize Devise Forgot Password Token In Rails"
date:   2020-04-21 09:00:00
permalink: customize-devise-forgot-password-token-in-rails
---

This is a documentation on how to change the user flow of forgot password using the [devise gem](https://github.com/heartcombo/devise).

## Motivation

I often work on projects that are purely API based and are served only on mobile devices as apps. This poses a problem when using devise because its main audience is web application. The user flow that it has set up thus assumes the presence and usage of web pages. That is absent for this case, and setting it up is troublesome to say the least.

In the case of forgot and resetting password flow, the user will receive an email with a link to reset their password. This link leads to a webpage and they will reset their password there and then.

For a pure API environment, this translates to an immense amount of extraneous work required. We need to setup the hosting of the webpages, prepare the styling assets, tweak css codes, wire up the SSL certificate, validate the input fields, and properly redirect from the website into the app once the password has been reset.

And the frustrating thing is that this is a small but essential function in a typical application. We cannot do away with it, but it is often neglected, or should I say taken for granted. And the disproportionally large effort it takes to set it up 1 web page just for this seemingly insignificant function is often overlooked.

I see the need to implement a way devise can allow user to change passwords without the use of webpage.

## Project Specifications

The UX flow in my projects will look like this.

The reset password flow will send an email with a token that the users will enter in their app upon submitting the form on forgot password.

The user will enter their new password as well as the token in their app to change their password.

## How Reset Password Work In Devise?

Before we can get to work, we need to understand how reset password flow runs in devise.

By default when the user submits his/her email in the forgot password flow, [this controller method in the Devise::PasswordsController](https://github.com/heartcombo/devise/blob/main/app/controllers/devise/passwords_controller.rb) is called.

```ruby
# POST /resource/password
def create
  self.resource = resource_class.send_reset_password_instructions(resource_params)
  yield resource if block_given?

  if successfully_sent?(resource)
    respond_with({}, location: after_sending_reset_password_instructions_path_for(resource_name))
  else
    respond_with(resource)
  end
end
The send_reset_password_instructions method is called on the class of the resource. The main code snippet is as shown below, retrieved from the [source code in devise github repository](https://github.com/heartcombo/devise/blob/769506e96cd45b36a311eeca293ce0228c58e5f3/lib/devise/models/recoverable.rb).

module Devise
  module Models
    module Recoverable
      extend ActiveSupport::Concern
      protected
        def set_reset_password_token
          raw, enc = Devise.token_generator.generate(self.class, :reset_password_token)

          self.reset_password_token   = enc
          self.reset_password_sent_at = Time.now.utc
          save(validate: false)
          raw
        end
      module ClassMethods
        def send_reset_password_instructions(attributes={})
          recoverable = find_or_initialize_with_errors(reset_password_keys, attributes, :not_found)
          recoverable.send_reset_password_instructions if recoverable.persisted?
          recoverable
        end
      end
    end
  end
end
```

Eventually, the function set_reset_password_token function will be called to generate the reset_password_token. This method will eventually become one of the User model’s instance methods when it includes the recoverable module.

The logic of generating the reset_password_token is wrapped in the TokenGenerator class of the Devise module as shown below.

```ruby
module Devise
  class TokenGenerator
    def generate(klass, column)
      key = key_for(column)

      loop do
        raw = Devise.friendly_token
        enc = OpenSSL::HMAC.hexdigest(@digest, key, raw)
        break [raw, enc] unless klass.to_adapter.find_first({ column => enc })
      end
    end
  end
end
```

As I want user to generate a token that they will need to enter together when they submit their new password after receiving it from an email, I definitely want to dictate the number of characters the token has for the sake of humane user experience. We will see how to tweak this method to generate a token of suitable length.

The function send_reset_password_instructions will also be triggered thereafter to send an email. By default, the reset password token that was generated will be appended to a url that is sent along in that email. That url meant for the users to click and go to a webpage to change their password. For my case, I will not be presenting the url to be clicked in the email, but just the the token string instead.

## Generate Custom Reset Password Token

Here we will change the set_reset_password_token for the User model. This will only affect the User model and not other models, which may be crucial for you.

In my projects, I usually have another devise model, ie. the AdminUser model who needs to access to a CMS system. The CMS system is authenticated by none other than devise in the usual devise way. Hence, I do not want to do a site way change to all my models due to this sort of “hybrid”.

Hence, the new code will look like this.

```ruby
protected
def set_reset_password_token
  raw, enc = Devise.token_generator.custom_generate(self.class, :reset_password_token)

  self.reset_password_token   = enc
  self.reset_password_sent_at = Time.now.utc
  save(validate: false)
  raw
end
```

The only line that was change is line 3. I am using the custom_generate method, which I define below.

```ruby
module Devise
  class TokenGenerator
    def custom_generate(klass, column)
      key = key_for(column)

      loop do
        raw = SecureRandom.alphanumeric(Rails.configuration.confirmation_token_length)
        enc = OpenSSL::HMAC.hexdigest(@digest, key, raw)
        break [raw, enc] unless klass.to_adapter.find_first({ column => enc })
      end
    end
  end
end
```
Compared to the above, the only line that is changed in this case is line 7. The default method uses Devise.friendly_token, the source code of which can be found here.

I replaced it with a custom method of mine to generate and alphanumeric string of a custom desired length.

Seeing that I change so little for each part of the code, I could have just redefined the Devise.friendly_token and save some effort in copying and pasting codes. However, due to the fact that I am still going to have an AdminUser that will make use of the default configuration of devise as it, I cannot apply it site wide. Of course, if I have only 1 Devise model to work with, that will be a plausible route to take.

## Send Email With Customized Reset Password Token

So now that the reset_password_token has been generated, it is time to send it out in the email.

There’s nothing to change on the Devise::Mailerclass here. All we need to change is the email view under reset_password_instructions.html.erb. Below is the default view from [devise repository](https://github.com/heartcombo/devise/blob/main/app/views/devise/mailer/reset_password_instructions.html.erb).

```erb
<p>Hello <%= @resource.email %>!</p>

<p>Someone has requested a link to change your password. You can do this through the link below.</p>

<p><%= link_to 'Change my password', edit_password_url(@resource, reset_password_token: @token) %></p>

<p>If you didn't request this, please ignore this email.</p>
<p>Your password won't change until you access the link above and create a new one.</p
We now have the @token that we can just display for users as a string, and we do not need edit_password_url link to be generated.
```

## Conclusion

This is how we can modify the reset_password_token using devise with the least possible code changes. It involves understanding the flow of logic throughout the different components in devise, as well as the role each component play, so that you know what can and should be modified. The same can be applied to the confirmation flow as well.

This can be integrated with doorkeeper and devise on top of this guide that I wrote on integrating these 2 gems without hiccups since the amount of changes is little and not show stopping.

