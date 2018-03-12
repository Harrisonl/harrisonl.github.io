---
layout:     post
title:      Pixyll has Pagination
title:      ETS - Async Reads & Writes
date:       2014-06-08 11:21:29
summary:    This post goes through how to build a basic async cache in elixir
categories: elixir ets
---


ETS is an exceptional tool that I feel is greatly under-used in the Elixir world.

[This talk](https://www.youtube.com/watch?v=D3IftRUQgqc) by Claudio from Erlang-Solutions highlights a very good implementation for achieving concurrent reads as well as sequential-concurrent writes with ETS.

{% highlight elixir %}
defmodule Cache do
  use GenServer

  @table :cache
  @name __MODULE__

  def start_link() do
    GenServer.start_link(@name, [], name: @name)
  end

  def init(_) do
    # :public and read concurrency are very important
    :ets.new(@table, [:named_table, :public, :read_concurrency])
    {:ok, []}
  end

  def get(key) do
    :ets.lookup(@table, key)
  end

  def set(key, val) do
    GenServer.cast(__MODULE__, {:set, key, val})
  end

  def handle_cast({:set, key, val}, state) do
    :ets.insert(@table, {key, val})
    {:noreply, state}
  end
end
{% endhighlight %}

Since we have `:public` and `:read_concurrency` set in our table options, **we have the ability to offload the reading of the ETS data to the calling process**, allowing the `Cache` genserver to focus on inserting only.

**However, you really need to understand your data for this to be effective as it really isn't suited to data that is going to be updated due to race conditions.**

For example, let's say you get a flood of data that needs to be written to the cache e.g. 10,000,000 struct-objects.

These objects contain a tracking counter key of some sort.

Let's that during these writes, a request comes in to increment counter `:foo`. So, something like this might happen:

{% highlight elixir %}
iex> foo = Cache.get(:foo)
iex> foo.counter
# => 10
iex> foo = %Struct{foo | counter: foo.counter + 1}
# => %Struct{counter: 11..}
iex> Cache.set(:foo, foo)
{% endhighlight %}

Nothing wrong with this - it all looks fine. The problem occurs when you look at what already is in the `Cache` mailbox.

Let's say that of those 10million write requests, req # 2,900,000 was to do the same thing - update `:foo` counter by one.

Since potentially this update would not have happened by the time the second update request comes through, `Cache.get(:foo)` is still going to return the old value of `10` whereas it really should be 11.

This means that the cache is going to get 2 write requests for `:foo` with a counter value of 11 rather than for 11 and the next for 12.
