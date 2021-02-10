---
uid: dispatchers
title: Dispatchers
---

# Dispatchers

## 调度器是做什么的?

Dispatchers负责调度ActorSystem内部运行的所有代码。Dispatchers是Akka.NET的重要组成部分，因为它们控制每个Actor的吞吐量和分享时间，从而为每个参与者提供公平的资源。

默认情况下，所有Actors共享一个全局调度器。除非您更改配置，否则此调度程序将在后台使用.NET线程池，该池针对大多数常见场景进行了优化。这意味着对于大多数情况，默认配置应该足够好。

#### 为什么要使用不同的调度器?

当消息到达Actor的邮箱时，dispatcher会安排消息的批量传递，并尝试在将线程释放给另一个Actor之前传递整批消息。虽然默认配置对于大多数场景来说已经足够好了，但是您可能需要（通过配置）更改调度器运行每个参与者所需的时间。

选择不同的调度器还有其他一些常见的原因。这些原因包括（但不限于）：

* 将一个或多个Actor隔离到特定线程，以便：
  * 确保高负载Actor不会因为消耗太多cpu时间而导致系统饥饿；
  * 确保重要的Actor总是有一个专门的线程来完成他们的工作；
  * 制造舱壁，确保系统某一部分产生的问题不会泄漏给其他部分；
* 允许Actor在特定的SyncrhonizationContext中执行；

> [!NOTE]
> 请考虑仅在特殊情况下使用自定义调度器。正确配置dispatchers需要了解框架的工作原理。自定义调度程序不应被视为性能问题的默认解决方案。对于复杂的应用程序来说，拥有一个或几个定制的调度器是正常的，对于系统中的大多数或所有参与者来说，需要定制的调度器配置是不常见的。

## 调度器和调度器配置

在本文档和大多数可用的Akka文献中，术语dispatcher用于指代dispatcher配置，但实际上它们是不同的东西。

- **调度器** 是负责在系统中调度代码执行的低级组件。这些组件内置于Akka.NET，它们的数量是固定的，您不需要创建或更改它们。

- **调度器 配置** 您可以创建自定义设置，以特定方式使用dispatchers。有一些内置的dispatcher配置，您可以根据需要为应用程序创建任意多个。

因此，当你读到“创建一个定制的调度器”时，它通常意味着“为一个内置的调度器使用一个定制的配置”.

## 配置 调度器

您可以使用HOCON配置节定义自定义调度器配置。

下面的示例创建了一个名为`my-dispatcher`的自定义调度器，可以在部署期间在一个或多个参与者中设置它：

```hocon
my-dispatcher {
    type = Dispatcher
    throughput = 100
    throughput-deadline-time = 0ms
}
```

然后可以使用部署配置设置actor的调度器：

```hocon
akka.actor.deployment {
    /my-actor {
        dispatcher = my-dispatcher
    }
}
```

或者你也可以在代码中配置：

```cs
system.ActorOf(Props.Create<MyActor>().WithDispatcher("my-dispatcher"), "my-actor");
```

#### Built-in Dispatcher Configurations

Some dispatcher configurations are available out-of-the-box for convenience. You can use them during actor deployment, [as described above](#configuring-dispatchers).

* **default-dispatcher** - A configuration that uses the [ThreadPoolDispatcher](#threadpooldispatcher). As the name says, this is the default dispatcher configuration used by the global dispatcher, and you don't need to define anything during deployment to use it.
* **internal-dispatcher** - To protect the internal Actors that is spawned by the various Akka modules, a separate internal dispatcher is used by default.
* **task-dispatcher** - A configuration that uses the [TaskDispatcher](#taskdispatcher).
* **default-fork-join-dispatcher** - A configuration that uses the [ForkJoinDispatcher](#forkjoindispatcher).
* **synchronized-dispatcher** - A configuration that uses the [SynchronizedDispatcher](#synchronizeddispatcher).

## Built-in Dispatchers

These are the underlying dispatchers built-in to Akka.NET:

### ThreadPoolDispatcher

It schedules code to run in the [.NET Thread Pool](https://msdn.microsoft.com/en-us/library/System.Threading.ThreadPool.aspx), which is ***good enough* for most cases.**

The `type` used in the HOCON configuration for this dispatcher is just `Dispatcher`.

```hocon
custom-dispatcher {
type = Dispatcher
throughput = 100
}
```

> [!NOTE]
> While each configuration can have it's own throughput settings, all dispatchers using this type will run in the same default .NET Thread Pool.

### TaskDispatcher

The TaskDispatcher uses the [TPL](https://msdn.microsoft.com/en-us/library/dd460717.aspx) infrastructure to schedule code execution. This dispatcher is very similar to the Thread PoolDispatcher, but may be used in some rare scenarios where the thread pool isn't available.

```hocon
custom-task-dispatcher {
  type = TaskDispatcher
  throughput = 100
}
```

### PinnedDispatcher

The `PinnedDispatcher` uses a single dedicated thread to schedule code executions. Ideally, this dispatcher should be using sparingly.

```hocon
custom-dedicated-dispatcher {
  type = PinnedDispatcher
}
```

### ForkJoinDispatcher

The ForkJoinDispatcher uses a dedicated threadpool to schedule code execution. You can use this scheduler isolate some actors from the rest of the system. Each dispatcher configuration will have it's own thread pool.

This is the configuration for the [*default-fork-join-dispatcher*](#built-in-dispatcher-configurations).  You may use this as example for custom fork-join dispatchers.

```hocon
default-fork-join-dispatcher {
  type = ForkJoinDispatcher
  throughput = 100
  dedicated-thread-pool {
	  thread-count = 3
	  deadlock-timeout = 3s
	  threadtype = background
  }
}
```

* `thread-count` - The number of threads dedicated to this dispatcher.
* `deadlock-timeout` - The amount of time to wait before considering the thread as deadlocked. By default no timeout is set, meaning code can run in the threads for as long as they need. If you set a value, once the timeout is reached the thread will be aborted and a new threads will take it's place. Set this value carefully, as very low values may cause loss of work.
* `threadtype` - Must be `background` or `foreground`. This setting helps define [how .NET handles](https://msdn.microsoft.com/en-us/library/system.threading.thread.isbackground.aspx) the thread.

### SynchronizedDispatcher

The `SynchronizedDispatcher` uses the *current* [SynchronizationContext](https://msdn.microsoft.com/en-us/magazine/gg598924.aspx) to schedule executions.

You may use this dispatcher to create actors that update UIs in a reactive manner. An application that displays real-time updates of stock prices may have a dedicated actor to update the UI controls directly for example.

> [!NOTE]
> As a general rule, actors running in this dispatcher shouldn't do much work. Avoid doing any extra work that may be done by actors running in other pools.

This is the configuration for the [*synchronized-dispatcher*](#built-in-dispatcher-configurations).  You may use this as example for custom fork-join dispatchers.

```hocon
synchronized-dispatcher {
  type = "SynchronizedDispatcher"
  throughput = 10
}
```

In order to use this dispatcher, you must create the actor from the synchronization context you want to run-it. For example:

```csharp
private void Form1_Load(object sender, System.EventArgs e)
{
  system.ActorOf(Props.Create<UIWorker>().WithDispatcher("synchronized-dispatcher"), "ui-worker");
}
```

#### Common Dispatcher Configuration

The following configuration keys are available for any dispatcher configuration:

* `type` - (Required) The type of dispatcher to be used: `Dispatcher`, `TaskDispatcher`, `PinnedDispatcher`, `ForkJoinDispatcher` or `SynchronizedDispatcher`.
* `throughput` - (Required) The maximum # of messages processed each time the actor is activated. Most dispatchers default to `100`.
* `throughput-deadline-time` - The maximum amount of time to process messages when the actor is activated, or `0` for no limit. The default is `0`.

> [!NOTE]
> The throughput-deadline-time is used as a *best effort*, not as a *hard limit*. This means that if a message takes more time than the deadline allows, Akka.NET won't interrupt the process. Instead it will wait for it to finish before giving turn to the next actor.

## Dispatcher aliases

When a dispatcher is looked up, and the given setting contains a string rather than a dispatcher config block, 
the lookup will treat it as an alias, and follow that string to an alternate location for a dispatcher config.
If the dispatcher config is referenced both through an alias and through the absolute path only one dispatcher will
be used and shared among the two ids.
