---
layout: post
category: programming
tags: ruby rails jwt auth knock gem token api
date: 2020-06-17 17:15:00
updated: 2020-06-24 09:45:00
description: >-
  How to set up and use the knock gem to add JWT auth to your Rails 6 API.
title: JWT Auth in Rails 6 with Knock
---

This post explains how to set up and use the [knock gem][1] for JWT auth in
your Rails 6 API. Currently, the whole knock [situation][2] is a [bit][3]
[confusing][4]. From what I can tell, a new maintainer took over (thank you!)
and is trying to get a solid release out the door, the first in three years.
Unfortunately, the existing docs and blog posts available on knock are
sometimes unclear or outdated. But I figured out how to get things working in
Rails 6, and now you can, too.

<!-- more -->

## Installation

At the time of this writing, the new release, 2.2, hasn't been pushed out to
RubyGems. So what you want to do is utilize the [unofficial 2.2 release][2]
by adding `knock` to your `Gemfile` in this way:

```ruby
# Add JWT authentication
# Need to use this specific commit, which is unofficially the 2.2 release,
#   because the new version hasn't been released to RubyGems yet.
gem "knock", github: "nsarno/knock", branch: "master",
    ref: "9214cd027422df8dc31eb67c60032fbbf8fc100b"
```

Once you've added that, go ahead and install:

```shell
bundle install
```

## The Model

Presumably, you have a `User` model for use with auth. For this part, [the
official docs][5] are solid and I recommend following them. I believe the docs
also have some info on what to do if your model is namespaced, or maybe even
named differently, though I haven't tried any of that stuff out.

The main requirement is that you either use `has_secure_password` in your
`User` model, or, alternatively, implement an `authenticate` method that does
the same sort of thing that the `authenticate` method added by
`has_secure_password` does (see [the docs for that method][6]). For many
people, adding `has_secure_password` like this will be all you need:

```ruby
class User < ApplicationRecord
  has_secure_password
  # Other stuff can be in this class, of course
end
```

The default setup with `has_secure_password` assumes that users will be
authenticating with an `email` and a `password` and not doing anything fancy
to authenticate other than checking the validity of the email/password
combination. **If this isn't the case, you need to take further steps.**
Otherwise, you can skip this next section.

### Further Steps for Non-Default Auth

You may want to have auth occur with a `username` and `password`, or something
similar, instead of the default `email` and `password`. In that case, you need
to override `self.from_token_request(request)` in your `User` model. For
instance, if you want to use a `username` instead of an `email` you'll need
something like this:

```ruby
class User < ApplicationRecord
  def self.from_token_request(request)
    User.find_by(name: request.params[:auth][:username])
  end
end
```

The reason you need to override this method is because the default
`self.from_token_request(request)` looks up the authenticating user by `email`.
The above version causes the user to be looked up by `username` instead.

For my purposes, I had to implement my own `authenticate` method as well as
`self.from_token_request` as mentioned in [the official docs][5], because I
have an unusual situation where my API actually authenticates with _another_
API for its login process.

Basically, you will want to override `authenticate(password)` if you want
your authentication to entail something other than simply checking the user's
password:

```ruby
class User < ApplicationRecord
  def authenticate(password)
    # Do your custom authentication here.
    # Return `true` if the auth should succeed, or `false` if it should fail.
  end
end
```

Without sharing too much private code, my override looks something like
this:

```ruby
class User < ApplicationRecord
  def authenticate(password)
    # ... I do a few things up here, then...
    if login # `login` is a custom method of mine, not a knock thing.
      self.last_logged_in = Time.now # Another custom thing of mine.
      save # This returns true if the updates I made to the user succeed.
    else
      false # This causes the auth to fail.
    end
  end
end
```

You don't have to make modifications to your user in your
`authenticate(password)`. I do, but you don't have to. All you need to do
is make sure your `authenticate(password)` returns either `true` or `false`.

## The Controllers

You need to create a controller for knock. Please note that if you have
any issues, try naming your controllers and routes exactly like mine.

In `controllers/`, create a `user_token_controller.rb` file with these
contents:

```ruby
class UserTokenController < Knock::AuthTokenController
end
```

That's right: empty controller. The `Knock::AuthTokenController` that it
inherits from provides everything you need.

*Unless*, of course, you're using a non-standard auth setup [as mentioned
above](#further-steps-for-non-default-auth). In that case, you'll want to
override the `auth_params`. For instance, if your auth setup uses a `username`
instead of an `email`, your controller might look something like this:

```ruby
class UserTokenController < Knock::AuthTokenController
  private
  def auth_params
    # Without overriding the auth_params here, you get "unpermitted
    #   parameter" errors for username. The call seems to work anyway,
    #   but this eliminates the error message from your logs.
    params.require(:auth).permit(:username, :password)
  end
end
```

My app is namespaced, so my full controller actually looks like _this_:

```ruby
module Api
  module V1
    class UserTokenController < Knock::AuthTokenController
      private
      def auth_params
        params.require(:auth).permit(:username, :password)
      end
    end
  end
end
```

In that `permit(...)`, you should permit params you need for your
authentication process. Mine just takes a username and password.

Second, at the top of your `ApplicationController` in
`controllers/application_controller.rb`, add these lines:

```ruby
include Knock::Authenticable
before_action :authenticate_user # Optional, see below
```

That first line is required. The second line is optional. When you add it to
`ApplicationController`, it blocks access to all of your controllers for
any request that doesn't include a valid JWT token in its header -- except
for the `UserTokenController`. The `UserTokenController` inherits from
`Knock::AuthTokenController`, so it can be accessed without a JWT token for
the purpose of actually _getting_ the JWT token. It looks like knock is
smart enough to take care of that for you; you don't have to add any special
rules to `UserTokenController` to allow this special access for authentication.

## The Route

Add a route in `config/routes.rb` that looks like this:

```ruby
post "/auth", to: "user_token#create"
```

I confirmed that the first part, `"/auth"`, can be whatever you want.
`"/login"`, `"/user_token`, whatever. The second part is required though --
you need to point to that `create` action. You didn't have to write a `create`
action yourself, because it's automagically provided by
`Knock::AuthTokenController`.

## Configuration

You may need to configure knock via an initializer, which is a file you can
create at `config/initializers/knock.rb` with these contents:

```ruby
Knock.setup do |config|
end
```

This initializer can be given a few different options, as detailed in [the
docs][5]. You may need to add some of those.

My app ran fine locally in my development environment without any options, but
for production, I had to configure the signature key, like so:

```ruby
Knock.setup do |config|
  config.token_secret_signature_key = -> { Rails.application.credentials.read }
end
```

Without that line, I was getting a fatal error in my `log/production.log` that
began like this:

```
TypeError (no implicit conversion of nil into String)
```

*Locally, where my app seemed to work with an empty initializer, I tried
deleting that `knock.rb` inititalizer file entirely, and my app seemed to run
fine (locally) without it, so it doesn't seem that the file is required if you
don't need to configure anything.*

## Give It a Go

At this point, you should restart your Rails app. Then use a REST client
to see if things are working.

A `POST` request to your route (my path is `http://localhost:3000/api/v1/auth`)
might look like this:

```json
{
  "auth": {
    "email": "demouser@example.com",
    "password": "testingpassword123"
  }
}
```

Or if you have a non-standard auth setup [as mentioned
above](#further-steps-for-non-default-auth), your request body might need to
look different. For instance, if your setup uses a `username` instead of an
`email`, it might look like this:

```json
{
  "auth": {
    "username": "demouser",
    "password": "testingpassword123"
  }
}
```

From what I can tell, that `auth` object is necessary, something knock looks
for. I didn't define it anywhere; knock just expects your params to be wrapped
in it.

Don't forget to include a header with `Content-Type` set to `application/json`.

You should get back something like this:

```json
{
  "jwt": "eyJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1OTI1MDc4MjcsInN1YiI6NH0.0gTKH4rmDFvI-mZmHIB52CooUDIEYZjQ1aLnX0DVT6w"
}
```

Add that long string as a `Bearer` token to the auth header of further
requests, and you should be all set. Without this token, your requests should
return a `401 Unauthorized`.

Now, in your controllers, you can use `current_user` to access the auth'd user.

## Troubleshooting

If you're having any troubles with this post whatsoever, first try naming
your things exactly how I name mine. Everything. I'm not sure -- since I
just figured this stuff out -- but I think some things might need to be named
in a certain way, at least by default.

Second, take a closer look at the [official docs][5] and the configuration
options [I mention above](#configuration).

Figuring this stuff out today was a bit of a drag, and I know there are other
people out there who are frustrated, going through the same thing. Knock is
great but it's in a bit of a weird place right now. Don't hesitate to
[email me][7] with any questions you have, and I'll try to point you in the
right direction.

## Feedback

Questions, comments, or tips for me? See a mistake in this post? [Send me an
email.](mailto:hello@davidgay.org)


[1]: https://github.com/nsarno/knock
[2]: https://github.com/nsarno/knock/pull/248
[3]: https://github.com/nsarno/knock/issues/250
[4]: https://github.com/nsarno/knock/issues/249
[5]: https://github.com/nsarno/knock/blob/master/README.md
[6]: https://api.rubyonrails.org/classes/ActiveModel/SecurePassword/ClassMethods.html#method-i-has_secure_password
[7]: mailto:hello@davidgay.org
