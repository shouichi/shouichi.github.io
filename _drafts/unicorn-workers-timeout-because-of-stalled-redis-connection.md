---
layout: post
title: Unicorn Workers Timeout Because of Stalled Redis Connection
---

I encountered a strange problem of unicorn workers timeout this week. So I want
to describe 1. how did I find the cause was Redis, 2. how did I fixed the
problem in this post.

# Prelude

The story started when timeout errors were found in the unicorn log.  It seemed
that workers were killed due to timeout by the master process while they were
waiting for something to complete.

{% highlight text %}
E, [2015-01-28T09:31:04.189396 #12083] ERROR -- : worker=3 PID:4513 timeout (61s > 60s), killing
E, [2015-01-28T09:31:36.365409 #12083] ERROR -- : worker=2 PID:2741 timeout (61s > 60s), killing
E, [2015-01-28T10:20:55.787742 #12083] ERROR -- : worker=6 PID:10447 timeout (61s > 60s), killing
{% endhighlight %}

# Finding The Cause

The first problem was that they were killed without any log. So we patched
unicorn and modified its config to dump the stack trace when workers were
killed. The idea is using different signal `INT` instead of `KILL` to kill
workers, trap the `INT` and dump the strack trace when killed.

{% highlight ruby %}
# config/initializers/unicorn_http_server_patch.rb
class Unicorn::HttpServer
  def kill_worker(signal, wpid)
    Process.kill(:INT, wpid)
    rescue Errno::ESRCH
      worker = WORKERS.delete(wpid) and worker.close rescue nil
  end
end
{% endhighlight %}

{% highlight ruby %}
# config/unicorn.rb
after_fork do |server, worker|
  trap(:INT) do
    File.open("#{Rails.root}/log/killed-workers-stack-trace.log", "a") do |file|
      file.puts Thread.current.backtrace.join("\n")
    end
    Process.kill :KILL, $$
  end
end
{% endhighlight %}

Thanks to this patch I could get the stack trace.

{% highlight text %}
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/connection/ruby.rb:238:in `call'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/connection/ruby.rb:238:in `write'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/connection/ruby.rb:238:in `write'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:221:in `block in write'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:206:in `io'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:220:in `write'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:184:in `block (3 levels) in process'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:178:in `each'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:178:in `block (2 levels) in process'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:295:in `ensure_connected'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:177:in `block in process'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:256:in `logging'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:176:in `process'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:147:in `call_pipelined'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:121:in `block in call_pipeline'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:243:in `with_reconnect'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis/client.rb:119:in `call_pipeline'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis.rb:2077:in `block in multi'
/home/deploy/app/shared/bundle/ruby/2.0.0/gems/redis-3.0.4/lib/redis.rb:36:in `block in synchronize'
{% endhighlight %}

# References

[Redis timeout/keepalive](http://librelist.com/browser//sidekiq/2012/9/27/redis-timeout-keepalive/)
