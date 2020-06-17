---
layout: post
category: software
tags: ruby rails jwt auth knock gem token api
date: 2020-06-17 17:15:00
description: >-
  How to set up and use the knock gem to add JWT auth to your Rails 6 API.
title: JWT Auth in Rails 6 with Knock
---

This post explains how to set up and use the [knock gem][1] for JWT auth in
your Rails 6 API. Currently, the whole knock [situation][2] is a [bit][3]
jacked [up][4]. From what I can tell, a new maintainer took over (thank you!)
and is trying to get a solid release out the door, the first in three years.
Unfortunately, the existing docs and blog posts available on knock are often
unclear and outdated. But I figured out how to get things working in Rails 6,
and now you can, too.

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
`has_secure_password` does (see [the docs for that method][6]). For most
people, adding `has_secure_password` like this will be all you need:

```ruby
class User < ApplicationRecord
  has_secure_password
  # Other stuff can be in this class, of course
end
```

For my purposes, I had to implement my own `authenticate` method and my own
`self.from_token_request` as mentioned in [the official docs][5], because I
have an unusual situation where my API actually authenticates with _another_
API for its login process. But since that's unusual, I'll explain that in
a forthcoming blog post, rather than at this moment. I'll update this post with
a link to that post when I write it. If you need help now, [email me][7].

## The Controllers

You need to create a controller for knock. Please note that if you have
any issues, try naming your controllers and routes exactly like mine.

In `controllers/`, create a `user_token_controller.rb` file with these
contents:

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
        # Without overriding the auth_params here, you get "unpermitted
        #   parameter" errors for username. The call seems to work anyway,
        #   but this eliminates the error message from your logs.
        params.require(:auth).permit(:username, :password)
      end
    end
  end
end
```

In that `permit(...)`, you should permit params you need for your
authentication process. Mine just takes a username and password.

You'll notice that this controller looks basically empty, and that's because
it is. The `Knock::AuthTokenController` that it inherits from provides
everything you need.

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

## Give It a Go

At this point, you should restart your Rails app. Then use a REST client
to see if things are working.

A `POST` request to your route (my path is `http://localhost:3000/api/v1/auth`)
might look like this:

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

Add that as a `Bearer` token to the auth header of further requests, and you
should be all set. Without this token, your requests should return a `401
Unauthorized`.

## Troubleshooting

If you're having any troubles with this post whatsoever, first try naming
your things exactly how I name mine. Everything. I'm not sure -- since I
just figured this stuff out -- but I think some things might need to be named
in a certain way, at least by default.

If that doesn't help, try adding a file, `config/initializers/knock.rb`,
with these contents:

```ruby
Knock.setup do |config|
end
```

This initializer can be given a few different options, as detailed in [the
docs][5]. You may need to add some of those.

I don't use any options, and so my initializer is empty. And I just tested
deleting the `knock.rb` inititalizer file entirely, and my app seems to run
fine without it, so it doesn't seem that the file is required if you don't need
to configure anything.

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
