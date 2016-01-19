---
layout: page
title: 并发
category: advanced
order: 3
lang: cn
---

Elixir 的一大卖点就是对并发的支持。得益于 ErlangVM，Elixir 的并发要比预期中简单得多。这个并发模型的基础是 Actors：通过消息传递来交互的进程（译者注：这个进程不是通常所说的操作系统级别的进程，可以理解为 ErlangVM 自己管理的轻量级进程）。

这节课，我们会讲 Elixir 自带的并发模型。在后面的章节中，我们还会介绍底层的实现机制：OTP 行为（behaviors）。

# 目录
- [进程](#section-1)
  - [消息传递](#section-2)
  - [进程链接](#section-3)
  - [进程监控](#section-4)

- [Agents](#agents)
- [Tasks](#tasks)

# 进程
ErlangVM 的进程很轻量级，可以运行在所有 CPU 上。看起来有点像原生的线程，但是它们更简单，而且同时运行几千个 Elixir 进程也是常事。

创建一个新进程最简单的方法是 `spawn`：它接受匿名函数或者命名函数作为参数。当你创建了一个新的进程，它会返回一个_进程标示符_，或者说 PID，在系统里来唯一确定这个进程。

我们来新建一个模块，然后定义一个要运行的函数：

```elixir
defmodule Example do
  def add(a, b) do
    IO.puts(a + b)
  end
end

iex> Example.add(2, 3)
5
:ok
```

要异步运行这个函数，我们可以使用 `spawn/3`：

```elixir
iex> spawn(Example, :add, [2, 3])
5
#PID<0.80.0>
```

## 消息传递
进程之间通信要依靠消息传递。有两个主要的组件做消息传递：`send/2` 和 `receive`。`send/2` 函数允许我们向 PIDs 发送消息，使用 `receive` 监听和匹配消息，如果没有匹配的消息，运行会一直处于不中断的状态。

```elixir
defmodule Example do
  def listen do
    receive do
      {:ok, "hello"} -> IO.puts "World"
    end
  end
end

iex> pid = spawn(Example, :listen, [])
#PID<0.108.0>

iex> send pid, {:ok, "hello"}
World
{:ok, "hello"}

iex> send pid, :ok
:ok
```

## 进程链接
当进程崩溃的时候，`spawn` 就会有问题（译者注：父进程不知道子进程出错会导致程序异常）。为了解决这个问题，我们可以用 `spawn_link` 把进程链接起来。两个链接起来的进程能收到相互的退出通知。

```elixir
defmodule Example do
  def explode, do: exit(:kaboom)
end

iex> spawn(Example, :explode, [])
#PID<0.66.0>

iex> spawn_link(Example, :explode, [])
** (EXIT from #PID<0.57.0>) :kaboom
```

有时候我们不希望链接的进程导致当前进程跟着崩溃，这时候就要捕捉进程的错误退出。当进程错误退出时，会向上层发送 `{:EXIT, from_pid, reason}` 三元组的消息。

```elixir
defmodule Example do
  def explode, do: exit(:kaboom)
  def run do
    Process.flag(:trap_exit, true)
    spawn_link(Example, :explode, [])

    receive do
      {:EXIT, from_pid, reason} -> IO.puts "Exit reason: #{reason}"
    end
  end
end

iex> Example.run
Exit reason: kaboom
:ok
```

## 进程监控
如果我们不想链接两个进程，但仍然希望能有错误信息通知呢？要做到这个，我们可以使用 `spawn_monitor`。当我们监控一个进程的时候，被监控进程崩溃的时候我们会接收到消息，而且不需要去捕获异常，也不会导致当前进程崩溃。

```elixir
defmodule Example do
  def explode, do: exit(:kaboom)
  def run do
    {pid, ref} = spawn_monitor(Example, :explode, [])

    receive do
      {:DOWN, ref, :process, from_pid, reason} -> IO.puts "Exit reason: #{reason}"
    end
  end
end

iex> Example.run
Exit reason: kaboom
:ok
```

# Agents
Agents 是后台运行的可以保存状态进程的抽象，我们可以在应用和节点中的进程中获取它的状态。Agent 的状态被设置成函数的返回值：

```elixir
iex> {:ok, agent} = Agent.start_link(fn -> [1, 2, 3] end)
{:ok, #PID<0.65.0>}

iex> Agent.update(agent, fn (state) -> state ++ [4, 5] end)
:ok

iex> Agent.get(agent, &(&1))
[1, 2, 3, 4, 5]
```

如果我们给 Agent 命名，后面就可以用名字而不是 PID 来指代它：

```elixir
iex> Agent.start_link(fn -> [1, 2, 3] end, name: Numbers)
{:ok, #PID<0.74.0>}

iex> Agent.get(Numbers, &(&1))
[1, 2, 3]
```

# Tasks
Tasks 提供了一种方式在后台执行一个函数，并且可以后面再获取它的返回值。在处理耗时操作的时候，tasks 会很有用，因为它们不阻塞当前的程序。

```elixir
defmodule Example do
  def double(x) do
    :timer.sleep(2000)
    x * 2
  end
end

iex> task = Task.async(Example, :double, [2000])
%Task{pid: #PID<0.111.0>, ref: #Reference<0.0.8.200>}

# Do some work

iex> Task.await(task)
4000
```