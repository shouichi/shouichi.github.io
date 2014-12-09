---
layout: post
title: API Versioning Using Rails Engine
---

# Abstract

API versioning is needed in terms of development. It is often case that the
only part of the API changes and remaining part stays the same in the next
version. Copy and paste works of course but it is needed to update multiple
versions when fixing bugs. Though branching is better than copy and paste, it
is tedious and error prone. In this post I explain the way to share code
between versions using rails engine.

# Methods

Given API v1 provides basic CRUD of user resources.

{% highlight ruby %}
Api::V1::Engine.routes.draw do
  resources :users
end
{% endhighlight %}

{% highlight ruby %}
module Api
  module V1
    class UsersController < ApplicationController
      def index
        @users = User.all
      end

      def show ...
      def create ...
      def update ...
      def destroy ...
    end
  end
end
{% endhighlight %}

{% highlight ruby %}
json.array! @users, partial: 'api/v1/users/user', as: :user
{% endhighlight %}

As you can see, v1 `index` API does not provide paging.

{% highlight bash %}
$ curl localhost:3000/api/v1/users.json
{"id":1,"name":"0"},
{"id":2,"name":"1"},
...
{% endhighlight %}

Now you decided to provide paging for v2 `index` API (which is incompatible
with v1) but the remaining API (`show`, `create`, `update` and `destroy`) stays
the same. In this case you can take advantage of rails engine and routes.

{% highlight ruby %}
Api::V2::Engine.routes.draw do
  # Define index action only.
  resources :users, only: [:index]

  mount Api::V1::Engine, at: '/'
end
{% endhighlight %}

The key point is mounting the previous version at the __bottom__. That way when
the request matches the current version routes, it hides the previous version,
otherwise it automatically falls back to the previous version.

{% highlight ruby %}
module Api
  module V2
    class UsersController < ApplicationController
      def index
        @users = User.page(params[:page])
      end
    end
  end
end
{% endhighlight %}

{% highlight ruby %}
json.users @users, partial: 'api/v1/users/user', as: :user
json.metadata do
  json.page do
    json.next users_path(page: @users.next_page)
  end
end
{% endhighlight %}

As you can see below, v2 `index` provides paging information. At the same time
`show` API (which is internally routed to v1) is also available under `/v2`
endpoint.

{% highlight bash %}
$ curl localhost:3000/api/v2/users.json
{
  "users":[
    {"id":1,"name":"0"},
    {"id":2,"name":"1"},
    ...
  ],
  "metadata":{
    "page":{
      "next":"/api/v2/users?page=2"
    }
  }
}

$ curl localhost:3000/api/v2/users/1.json
{"id":1,"name":"0"}
{% endhighlight %}

# Conclusion

You can share most of the code between versions using rails engine. There are
two key points.

1. Define routes only you want to override.
1. Mount the previous version at the bottom.

The working example is available at [github](https://github.com/shouichi/rails-api-versioning-example).
Thanks for reading, happy hacking!

# References

- [Getting Started with Engines](http://guides.rubyonrails.org/engines.html)
- [Rails Routing from the Outside In](http://guides.rubyonrails.org/routing.html)
