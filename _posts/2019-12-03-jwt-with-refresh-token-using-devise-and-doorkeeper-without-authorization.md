---
title:  "JWT With Refresh Token Using Devise And Doorkeeper Without Authorization"
date:   2019-12-03 09:00:00
permalink: jwt-with-refresh-token-using-devise-and-doorkeeper-without-authorization
---

This is a documentation on setting up the authentication system of a rails project in a primarily API environment.

Rails is essentially a framework for bootstrapping applications on the web environment. The support for APIs is thus lacking. One aspect of it is an off the shelf authentication system that can fit both the API and web environment on the same monolith application.

The [Devise gem](https://github.com/plataformatec/devise), while hugely popular and has established itself as the de facto authentication gem in the Rails world, does not come supported with an authentication system fit for interaction via APIs. The main reason is because it relies on cookies, which is strictly a browser feature.

To overcome this, often, we have to use other gems to couple with it to leverage on its scaffolded features for user authentications.

In this article, we will use the Doorkeeper and Devise combination to provide an authentication using [JSON Web Tokens (JWT)](https://en.wikipedia.org/wiki/JSON_Web_Token), the modern day best practices for authentication via APIs.

But let us first understand what kind of authentication system we are building and why we choose Doorkeeper.

## The Example Authentication System

Now, as a disclaimer, there are many ways to setup an authentication system.

One such consideration is the [devise-jwt gem](https://github.com/waiting-for-dev/devise-jwt), which serves as a direct replacement to the cookies for your APIs. It is simple to implement and allows you to choose from multiple strategies to expire your token. Except that it does not come with a [refresh token](https://github.com/waiting-for-dev/devise-jwt/issues/7).

This implies that the token will expire and the user will have to login again. If your application requires such security, you can consider this gem instead.

However, in this article, the authentication system that I will like to set up is one that allows user to log in via JWT that will expire, and upon expiry, the front end can use the refresh token to get a new JWT without having the user to login again. This allows the user to stay logged in without compromising security excessively.

Why do we need to ensure the JWT expires?

## Security Considerations Using JWT

Allowing user to be logged in permanently is kind of the standard user flow for many applications nowadays. The easiest way to implement this is to not expire the JWT. However, that is a recipe for disaster. It is akin to passing your password around when making API requests. And the moment it gets compromised, malicious attackers can have all the time in the world to explore your account and even plan their attacks, and leaving the users all the time in the world to say their prayers.

We thus have to enforce expiry on the JWT at the very least. To accomplish that without forcing the user to have to login again is to use a refresh token.

A refresh token stays in the local machine for the whole of it lifetime, or until the user actively logs out. This allows that the access token, which is dispatched out into the wild wild west otherwise known as the Internet, can at least expire within a certain period of time. And when it expires, the front end can use the refresh token to get a new access token to allow the user to continue its current session as though he or she is still logged in. So even if the access token gets compromised in the world beyond the walls, the potential damage is reduced.

This mechanism is made into a [standard known as Oauth](https://en.wikipedia.org/wiki/OAuth). There are many libraries out there that implements this already, and it is widely adopted among many of the software products that we use like Google account, facebook and twitter.

However, while this works with authenticating with these external providers, it has a crucial requirement that we do not want when implementing our own in house authentication system (I am referring to the old school email and password login). That step is the authorization step.

Some of us may have come across such a  request when we try to sign up with an app via Facebook, as shown below:

INSERT IMAGE oauth authorization

While this feature is absolutely essential in the OAuth protocol. it presents an awkwardness when we want to leverage on the OAuth libraries to implement JWT with refresh token for our in house authentication.

## The Awkwardness Of OAuth

Just make sure we are on the same page, here are a summary of the points that led up to this awkwardness.

First, we need to make the tokens expire for security reasons.

Second, refresh token are here to the rescue, and they are used in the OAuth protocol.

Third, unfortunately, OAuth requires an authorization step, which in house authentication system do not need.

Last, we cannot leverage on the various OAuth implementation out there to implement a JWT with refresh token without having to hack these libraries and somehow sidestep the authorization step.

## Hacking Doorkeeper

The OAuth library that we will be using is Doorkeeper. Its wiki page already has a [section on skipping the authorization step](https://doorkeeper.gitbook.io/guides/configuration/skip-authorization), which certainly signals the demand for such an implementation. However, there are some points missing from this implementation and this article will try to cover more of them. These steps are highly influenced by [this blog post](https://naturaily.com/blog/api-authentication-devise-doorkeeper-setup).

First, install doorkeeper and its migration files, following its instructions.

```bash
rails g doorkeeper:install
rails g doorkeeper:migration
```

## Changes To The Migration Files

Edit the migration file like this.

```ruby
# frozen_string_literal: true

class CreateDoorkeeperTables < ActiveRecord::Migration[6.0]
  def change
    create_table :oauth_access_tokens do |t|
      t.references :resource_owner, index: true
      t.integer :application_id
      t.text :token, null: false
      t.string :refresh_token
      t.integer :expires_in
      t.datetime :revoked_at
      t.datetime :created_at, null: false
      t.string :scopes
    end

    # required to allow model.destroy to work
    create_table :oauth_access_grants do |t|
      t.references :resource_owner, null: false
      t.integer :application_id
      t.string   :token, null: false
      t.integer  :expires_in, null: false
      t.text     :redirect_uri, null: false
      t.datetime :created_at, null: false
      t.datetime :revoked_at
      t.string   :scopes, null: false, default: ''
    end

    # Uncomment below to ensure a valid reference to the resource owner's table
    add_foreign_key :oauth_access_tokens, :users, column: :resource_owner_id
  end
end
```

Compared to the original generated copy of the migration file, we have removed the oauth_applications table which refers to the application that we want to grant permission to in the authorization step. Since we are skipping the authoirzation, there is no need to have this unused table.

Next we have changed

```ruby
t.references :application, null: false
```

into

```ruby
t.integer :application_id
```

Since the table is no longer present, we cannot use the references helper, and need to resort to specifying the the basic data type. We are still keeping this column in the database although we have deleted the application table because Doorkeeper uses this attribute while running its operation. Without it, an error will occur along the lines of “column not found“.

In fact, we also do not need the oauth_access_grants table, which is the bridge between the oauth_access_tokens table and the oauth_applications. It records which token authorized which application. However, without it, an error will be thrown when destroying a user record from the database. If you do not have such a feature, feel free to remove this table as well.

Lastly, only keep the foreign key implementation on oauth_access_tokens and change the model name according to whatever you have named your model.

## Changes To The Initializer File

Edit the configuration in the doorkeeper initializer file as such:

```ruby
# frozen_string_literal: true

Doorkeeper.configure do
  ...
  resource_owner_from_credentials do |routes|
    user = User.find_for_database_authentication(email: params[:email])
    request.env['warden'].set_user(user, scope: :user, store: false)
    user
  end
  ...
  use_refresh_token
  ...
  grant_flows %w[password]
  ...
  skip_authorization do
    true
  end
  ...
  api_only
  base_controller 'ActionController::API'
end
```

We are essentially following [this documentation on their wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Using-Resource-Owner-Password-Credentials-flow), but with some additional content and some slight changes, to implement an authentication flow whereby the token is returned in exchange for the credentials of the resource owner, in this case the user’s email and password.

Line 5 to 9 is the main implementation.

On line 6, we are instructing Doorkeeper to use Devise method, find_for_database_authentication, for authenticating the correct user. This method will run use the underlying warden gem in Devise to do its authentication magic. This, however, will save the user in the session, which can be a problem when we check for sessions in the controller level. More on this later. We undo this in line 7.

On line 7, we instruct warden to set the user only for the request and not store it in the session, as documented [here](https://github.com/wardencommunity/warden/wiki/users).

On line 11, uncomment use_refresh_token to ensure a refresh token is generated on login.

Line 13 is for older version of Doorkeeper at 2.1+. [More information in the above mentioned wiki page](https://github.com/doorkeeper-gem/doorkeeper/wiki/Using-Resource-Owner-Password-Credentials-flow).

Line 15 to 17, we instruct Doorkeeper to skip the authorization step.

Line 19, we set mode to api_only. This can help to optimize the application to a certain extent. For example, it skips forgery protection checks that is not necessary in an API environment, which reduces computational requirement and latency.

Line 20, I am just explicitly setting the base controller to use ActionController::API instead of the default ActionController::Base, although this should have already been implemented when the mode is set to api_only.

## Controller Level

Devise comes with a helper method, current_user or whatever your model name is, to access the current authenticated resource. This, however, will return a nil value in the current implementation because the underlying method will not be working. The underlying method is, taken from the [source code](https://github.com/heartcombo/devise/blob/main/lib/devise/controllers/helpers.rb):

```ruby
def current_#{mapping}
  @current_#{mapping} ||= warden.authenticate(scope: :#{mapping})
end
```

With reference to [this stackoverflow answer](https://stackoverflow.com/questions/20766357/how-to-access-current-user-from-a-doorkeeper-authenticated-session/50776537#50776537), we will modify it to look like this:

```ruby
def current_user
  @current_user ||= if doorkeeper_token
                      User.find(doorkeeper_token.resource_owner_id)
                    else
                      warden.authenticate(scope: :user, store: false)
                    end
end
```

We have essentially overwritten the default implementation by Devise to check for the “current_user” using the doorkeeper_token first, and fallback on the default implementation. The fallback will be useful in the event where our application will still be using the traditional login methods via a web browser. Feel free to remove it if you are not going to have such any request coming from a web browser. And of course, remember to handle the scenario of a nil doorkeeper_token.

Last but not least, implement that authorization check at the correct routes and actions in the Doorkeeper::TokensController via the before_action callback like how you would when using just Devise alone.

before_action :doorkeeper_authorize!
Custom Controller
I personally have some custom code that I want to add to all my APIs so that when the frontend consumes my APIs, they will not be left stunned by responses having different JSON structure.

I keep a response_code and a response_message in all my APIs for the frontend to react accordingly and trigger the desired UX flow.

Here is how I modify my controller. Let’s start off with some modification to the Doorkeeper modules.

```ruby
module Doorkeeper
  module OAuth
    class TokenResponse
      def body
        {
          # copied
          "access_token" => token.plaintext_token,
          "token_type" => token.token_type,
          "expires_in" => token.expires_in_seconds,
          "refresh_token" => token.plaintext_refresh_token,
          "scope" => token.scopes_string,
          "created_at" => token.created_at.to_i,
          # custom
          response_code: 'custom.success.default',
          response_message: I18n.t('custom.success.default')
        }.reject { |_, value| value.blank? }
      end
    end
  end
end
```

Here, I [modify the response from Doorkeeper](https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-Token-Response) to add in my required keys. I am using [I18n](https://guides.rubyonrails.org/i18n.html) to handle the custom messages and prepare the application for a global audience.

Next, the error response. By default, Doorkeeper returns the keys error and error_description. That is different from what I want. I will overwrite it totally.

```ruby
module Doorkeeper
  module OAuth
    class ErrorResponse
      # overwrite, do not use default error and error_description key
      def body
        {
          response_code: "doorkeeper.errors.messages.#{name}",
          response_message: description,
          state: state
        }
      end
    end
  end
end
```

`name`, `description` and `state` are [accessible variables in the default class](https://github.com/doorkeeper-gem/doorkeeper/blob/master/lib/doorkeeper/oauth/error_response.rb). I integrate them into my custom API response for standardization purpose.

Now the controller. There are 3 main methods: login, refresh and logout. Let’s go through them.

```ruby
module Api
  module V1
    class TokensController < Doorkeeper::TokensController
      before_action :doorkeeper_authorize!, only: [:logout]

      def login
        user = User.find_for_database_authentication(email: params[:email])

        case
        when user.nil? || !user.valid_password?(params[:password])
          response_code = 'devise.failure.invalid'
          render json: {
            response_code: response_code,
            response_message: I18n.t(response_code)
          }, status: 400
        when user&.inactive_message == :unconfirmed
          response_code = 'devise.failure.unconfirmed'
          render json: {
            response_code: response_code,
            response_message: I18n.t(response_code)
          }, status: 400
        when !user.active_for_authentication?
          create
        else
          create
        end
      end

      def refresh
        create
      end

      def logout
        # Follow doorkeeper-5.1.0 revoke method, different from the latest code on the repo on 6 Sept 2019

        params[:token] = access_token

        revoke_token if authorized?
        response_code = 'custom.success.default'
        render json: {
          response_code: response_code,
          response_message: I18n.t(response_code)
        }, status: 200
      end

      private

      def access_token
        pattern = /^Bearer /
        header = request.headers['Authorization']
        header.gsub(pattern, '') if header && header.match(pattern)
      end
    end
  end
end
```

Firstly, I am applying the doorkeeper_authorize! callback on the logout method only as that is the only method that will require the user to be logged in.

The login method will largely follow what we defined in the initializer file under the resource_owner_from_credentials block. The modification here is to define specific error scenarios and their respective response_code here. For those scenarios that are of no interest to me, I will leave it to the catch-all case and and return what is now the default modified ErrorResponse.

The second case in particular is specific to my project. I allow admin users to create the users, and have a flag (created_by_admin_and_authenticated) to differentiate them.

nil means the user registered normally
false means they are created by the admin user, but have yet to authenticate with the email that our server sent out to them
true means they are created by admin user and have also authenticated their email address
I will force users who are created by admin users but have yet to authenticate via email to reset their password, leveraging on what Devise has already provided with its password module.

Note: this is definitely much to be optimized here. For example, the find_for_database_authentication method is being called twice here for a successful user login, once in this custom controller and the other in the default Doorkeeper::TokensController create method.

The refresh method to refresh the access_token is practically the same as the default create method, but I am overriding it here because I use [ApiPie](https://github.com/Apipie/apipie-rails) to add documentation to the routes. For those who do not use ApiPie, we define its required parameters, headers etc. above the line 31 to define the documentation for the refresh method. I also can rename the route in doing so to create an API that the front end developers that I am working with would find more familiar with.

The logout method makes use of the revoke_token method, according to its [source code](https://github.com/doorkeeper-gem/doorkeeper/blob/master/app/controllers/doorkeeper/tokens_controller.rb), to revoke the JWT.

In my application, I require my frontend to add the JWT token in the Authorization header instead of a parameter in the request body based on convention. Doorkeeper, on the other hand, expects the token to be present in the params. To overcome this, I created the custom private access_token method to get the token in the header that the front end has placed in their requests. That token is then placed in the params object behind the key named token as Doorkeeper would have expected. Doorkeeper can then do its thing without having to modify any of its internal workings.

Since the revoke_token method provided by Doorkeeper will make use of the token key in the params, I will first use the private access_token method to extract the JWT token from the Authorization header. Then add it as the value to the token key of the params variable.

The logout method is required for the front end to dispose of the current access token they have for security purposes. I also use it to remove the users’ devices token so that they do not receive push notifications after logging out.

## Login Request

```json
{
	"email": "user1@test.com",
	"password": "user1@test.com",
	"grant_type": "password"
}
```

[A login request will have these keys](https://github.com/doorkeeper-gem/doorkeeper/wiki/Using-Resource-Owner-Password-Credentials-flow). In particular, the grant_type strategy used should be password.

## Conclusion

You should be able to login with the correct credentials with the default Doorkeeper::TokensController and access your controllers with the correct resource, just like how you would when using Devise alone. Otherwise, you can use your custom controller inherit and customise the authentication routes, as I have demonstrated.

Hope this was helpful!

