---
layout: post
title: HTTP Routing in Finagle
date: 2016-04-23 04:53:11
---

{% highlight scala %}
import java.net.InetSocketAddress
import java.util.concurrent.TimeUnit

import com.twitter.finagle.Service
import com.twitter.finagle.builder.ServerBuilder
import com.twitter.finagle.http.Method.{Get, Patch}
import com.twitter.finagle.http.filter.ExceptionFilter
import com.twitter.finagle.http.path.{->, /, Integer, Root}
import com.twitter.finagle.http.service.RoutingService
import com.twitter.finagle.http.{Http, Request, Response}
import com.twitter.finagle.service.TimeoutFilter
import com.twitter.finagle.util.DefaultTimer
import com.twitter.util.{Await, Duration, Future}


object Router {
  def apply(): Service[Request, Response] = {
    RoutingService.byMethodAndPathObject {
      case Get -> Root / "users" / username => UserService(username)
      case Patch -> Root / "posts" / Integer(postId) => PostService(postId)
    }
  }
}

object UserService {
  def apply(username: String): Service[Request, Response] = {
    new Service[Request, Response] {
      override def apply(request: Request): Future[Response] = {
        val response = Response()
        response.setContentString(username)
        Future.value(response)
      }
    }
  }
}

object PostService {
  def apply(postId: Integer): Service[Request, Response] = {
    new Service[Request, Response] {
      override def apply(request: Request): Future[Response] = {
        val response = Response()
        response.setContentString(postId.toString)
        Future.value(response)
      }
    }
  }
}

object ServerMain {
  def main(args: Array[String]): Unit = {
    val service = new TimeoutFilter(Duration(5, TimeUnit.SECONDS), DefaultTimer.twitter) andThen
      ExceptionFilter andThen
      Router()

    val server = ServerBuilder()
      .codec(Http())
      .bindTo(new InetSocketAddress(8080))
      .name("example")
      .build(service)
    Await.result(server)
  }
}
{% endhighlight %}

{% highlight bash %}
$ curl -i localhost:8080/users/string
HTTP/1.1 200 OK
Content-Length: 6

string
$ curl -i localhost:8080/users/123
HTTP/1.1 200 OK
Content-Length: 3

123
$ curl -i -X POST localhost:8080/users/string
HTTP/1.1 404 Not Found
Content-Length: 0

$ curl -i -X PATCH localhost:8080/posts/123
HTTP/1.1 200 OK
Content-Length: 3

123

$ curl -i -X PATCH localhost:8080/posts/string
HTTP/1.1 404 Not Found
Content-Length: 0

{% endhighlight %}
