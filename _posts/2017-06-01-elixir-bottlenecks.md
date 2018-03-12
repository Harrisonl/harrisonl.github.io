---
layout:     post
title:      Quickly See which processes are hogging CPU
date:       2014-06-08 11:21:29
summary:    Tips and tools for determining bottlenecks in an Elixir application.
categories: elixir performance
---

Recently I was [watching a talk](https://www.youtube.com/watch?v=5SbWapbXhKo) from [Saša Jurić](http://theerlangelist.com/) which gives you a solid understanding of how the beam VM works. If you haven't watched it, I definitely recommend checking it out.

Anyway, in this talk, he shows a method for quickly finding resource-hogging processes in a live environment, all without bringing the system down.

Let's say you have a function that involves some sort of recursion, and somehow has ended up in an infinite loop of sorts.

In a live system, this is obviously bad and can result in the process crashing.

A simple way to:

a) Figure which process is causing this
b) Figure out what the hell is going on with the process

{% highlight elixir %}
iex> Process.list
|> Enum.map(&{Process.info(&1, :reductions), &1})
|> Enum.take(5)
# => {{:reductions, 301203128421}, #PID<0.82.0>}
# => {{:reductions, 292424}, #PID<0.32.0>}
# => {{:reductions, 2345}, #PID<0.24.0>}
# => {{:reductions, 232}, #PID<0.62.0>}
# => {{:reductions, 123}, #PID<0.58.0>}
{% endhighlight %}

Here we can clearly tell that `#PID<0.82.0>` has gone through quite a few reductions (method calls). This is pretty obviously our culprit process. Let's do some more digging and see what's going on here:

{% highlight elixir %}
# pid = #PID<0.82.0>
iex> :dbg.tracer
iex> :dbg.p(pid, [:call]) # trace calls
iex> :dbg.tpl(:_, []); :timer.sleep(1000); :dbg.stop();

# => call 'Elixir.Module.Function' :method(arg1, arg2..)
# => call 'Elixir.Module.Function' :method(arg1+2, arg2..)
{% endhighlight %}

Now we can see exactly what is going on in this process and which functions are being called (and with what arguments).

Hopefully this is enough information for us to go away and fix the bug - but unfortunately, this process is still alive and recursing, so we need to take care of that first.

{% highlight elixir %}
iex> Process.exit(pid, :kill) # similar to kill -9
{% endhighlight %}

Now ideally you wouldn't actually want to have to do this in a live environment however, it's nice to know the ability is there!
