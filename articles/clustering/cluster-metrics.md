---
uid: cluster-metrics
title: Akka.Cluster.Metrics module
---

# Akka.Cluster.Metrics module

集群的成员节点可以通过`Cluster Metrics`扩展收集系统健康指标，并将其发布到其他集群节点和系统事件总线上的注册者。

集群指标主要用于负载平衡路由器，还可以用于实现基于高级度量的节点生命周期，例如当CPU占用时间过长时“节点让它崩溃”。

状态为`WeaklyUp`的集群成员（如果启用了该功能）将参与集群指标的收集和分发。

## Metrics Collector

指标收集器功能接口为`Akka.Cluster.Metrics.IMetricsCollector`.

不同的收集器实现可以提供不同的指标发布到集群中。

当前支持的指标定义在 `Akka.Cluster.Metrics.StandardMetrics`类：

* `MemoryUsed`—分配给当前运行进程的总内存
* `MemoryAvailable`-可用内存
* `MaxMemoryRecommended`-如果已设置，则为当前进程建议内存限制
* `Processors`—可用处理器的数量
* `cprupAccessUsage`-当前进程的CPU使用率
* `CpuTotalUsage`-总CPU使用率

> 注意：目前，由于某些.NET核心限制，“CpuTotalUsage”与“cprupAccessUsage”指标相同，
> 但这是在不久的将来要解决的问题（见[本期](https://github.com/akkadotnet/akka.net/issues/4142)详细信息）。

它收集了上面定义的所有指标都在``Akka.Cluster.Metrics.Collectors.DefaultCollector`中实现了

您还可以插入自己的指标收集器实现。

默认情况下，指标扩展将使用`collector provider fall back`并尝试按以下顺序加载它们：
1. 已配置用户提供的收集器（有关详细信息，请参阅`Configuration`部分）
2. 内置 `Akka.Cluster.Metrics.Collectors.DefaultCollector`收集器

## Metrics Events

指标扩展定期将集群指标的当前快照发布到节点系统事件总线。

发布间隔由`akka.cluster.metrics.collector.sample-interval`setting。

`Akka.Cluster.Metrics.Events.ClusterMetricsChanged`事件将包含节点的最新指标以及在收集器采样间隔期间接收到的其他群集成员节点指标。
您可以向这些事件订阅侦听器参与者，以便实现自定义节点生命周期：
```c#
ClusterMetrics.Get(Sys).Subscribe(metricsListenerActor);
```

## Adaptive Load Balancing

`AdaptiveLoadBalancingPool`/`AdaptiveLoadBalancingGroup`根据集群指标数据对发送到集群节点的消息执行负载平衡。

它使用相应节点的剩余内存容量数据计算随机选择的路由对象的概率。

它可以配置为使用特定的'imericsselector'实现来生成概率，即权重：

* `memory`/`MEMORYMETRICSELECTOR`-已用内存和最大可用内存。基于剩余内存容量的权重：（max-used）/max

* `cpu`/`cpumetercselector`—以百分比表示的cpu利用率。基于剩余cpu容量的权重：1-利用率

* `mix`/`MixMetricsSelector`-结合了内存和cpu。基于组合选择器剩余容量平均值的权重。

The collected metrics values are smoothed with [exponential weighted moving average](https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average). 
In the cluster configuration you can adjust how quickly past data is decayed compared to new data.

Let’s take a look at this router in action. What can be more demanding than calculating factorials?

The backend worker that performs the factorial calculation:

[!code-csharp[RouterUsageSample](../../../src/core/Akka.Docs.Tests/Cluster.Metrics/RouterUsageSample.cs?name=FactorialBackend)]

The frontend that receives user jobs and delegates to the backends via the router:

[!code-csharp[RouterUsageSample](../../../src/core/Akka.Docs.Tests/Cluster.Metrics/RouterUsageSample.cs?name=FactorialFrontend)]

As you can see, the router is defined in the same way as other routers, and in this case it is configured as follows:

```
akka.actor.deployment {
  /factorialFrontend/factorialBackendRouter = {
    # Router type provided by metrics extension.
    router = cluster-metrics-adaptive-group
    # Router parameter specific for metrics extension.
    # metrics-selector = memory
    # metrics-selector = cpu
    metrics-selector = mix
    #
    routees.paths = ["/user/factorialBackend"]
    cluster {
      enabled = on
      use-roles = ["backend"]
      allow-local-routees = off
    }
  }
}
```

It is only `router` type and the `metrics-selector` parameter that is specific to this router, other things work in the same way as other routers.

The same type of router could also have been defined in code:

### Group Router

[!code-csharp[RouterInCodeSample](../../../src/core/Akka.Docs.Tests/Cluster.Metrics/RouterInCodeSample.cs?name=RouterInCodeSample1)]

### Pool Router

[!code-csharp[RouterInCodeSample](../../../src/core/Akka.Docs.Tests/Cluster.Metrics/RouterInCodeSample.cs?name=RouterInCodeSample2)]

## Subscribe to Metrics Events

It is possible to subscribe to the metrics events directly to implement other functionality.

[!code-csharp[MetricsListenerSample](../../../src/core/Akka.Docs.Tests/Cluster.Metrics/MetricsListenerSample.cs)]

## Custom Metrics Collector

Metrics collection is delegated to the implementation of `Akka.Cluster.Metrics.IMetricsCollector`.

You can plug-in your own metrics collector instead of built-in `Akka.Cluster.Metrics.Collectors.DefaultCollector`.

Custom metrics collector implementation class must be specified in the `akka.cluster.metrics.collector.provider` configuration property.

## Configuration

The Cluster metrics extension can be configured with the following properties:

```
##############################################
# Akka Cluster Metrics Reference Config File #
##############################################

# This is the reference config file that contains all the default settings.
# Make your edits in your application.conf in order to override these settings.

# Cluster metrics extension.
# Provides periodic statistics collection and publication throughout the cluster.
akka.cluster.metrics {

  # Full path of dispatcher configuration key.
  dispatcher = "akka.actor.default-dispatcher"

  # How long should any actor wait before starting the periodic tasks.
  periodic-tasks-initial-delay = 1s

  # Metrics supervisor actor.
  supervisor {

    # Actor name. Example name space: /system/cluster-metrics
    name = "cluster-metrics"

    # Supervision strategy.
    strategy {

      # FQCN of class providing `akka.actor.SupervisorStrategy`.
      # Must have a constructor with signature `<init>(com.typesafe.config.Config)`.
      # Default metrics strategy provider is a configurable extension of `OneForOneStrategy`.
      provider = "Akka.Cluster.Metrics.ClusterMetricsStrategy, Akka.Cluster.Metrics"
      
      # Configuration of the default strategy provider.
      # Replace with custom settings when overriding the provider.
      configuration = {

        # Log restart attempts.
        loggingEnabled = true

        # Child actor restart-on-failure window.
        withinTimeRange = 3s

        # Maximum number of restart attempts before child actor is stopped.
        maxNrOfRetries = 3
      }
    }
  }

  # Metrics collector actor.
  collector {

    # Enable or disable metrics collector for load-balancing nodes.
    # Metrics collection can also be controlled at runtime by sending control messages
    # to /system/cluster-metrics actor: `akka.cluster.metrics.{CollectionStartMessage,CollectionStopMessage}`
    enabled = on

    # FQCN of the metrics collector implementation.
    # It must implement `akka.cluster.metrics.MetricsCollector` and
    # have public constructor with akka.actor.ActorSystem parameter.
    # Will try to load in the following order of priority:
    # 1) configured custom collector 2) internal `SigarMetricsCollector` 3) internal `JmxMetricsCollector`
    provider = ""

    # Try all 3 available collector providers, or else fail on the configured custom collector provider.
    fallback = true

    # How often metrics are sampled on a node.
    # Shorter interval will collect the metrics more often.
    # Also controls frequency of the metrics publication to the node system event bus.
    sample-interval = 3s

    # How often a node publishes metrics information to the other nodes in the cluster.
    # Shorter interval will publish the metrics gossip more often.
    gossip-interval = 3s

    # How quickly the exponential weighting of past data is decayed compared to
    # new data. Set lower to increase the bias toward newer values.
    # The relevance of each data sample is halved for every passing half-life
    # duration, i.e. after 4 times the half-life, a data sample’s relevance is
    # reduced to 6% of its original relevance. The initial relevance of a data
    # sample is given by 1 – 0.5 ^ (collect-interval / half-life).
    # See http://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average
    moving-average-half-life = 12s
  }
}

# Cluster metrics extension serializers and routers.
akka.actor {

  # Protobuf serializer for remote cluster metrics messages.
  serializers {
    akka-cluster-metrics = "Akka.Cluster.Metrics.Serialization.ClusterMetricsMessageSerializer, Akka.Cluster.Metrics"
  }

  # Interface binding for remote cluster metrics messages.
  serialization-bindings {
    "Akka.Cluster.Metrics.Serialization.MetricsGossipEnvelope, Akka.Cluster.Metrics" = akka-cluster-metrics
    "Akka.Cluster.Metrics.AdaptiveLoadBalancingPool, Akka.Cluster.Metrics" = akka-cluster-metrics
    "Akka.Cluster.Metrics.MixMetricsSelector, Akka.Cluster.Metrics" = akka-cluster-metrics
    "Akka.Cluster.Metrics.CpuMetricsSelector, Akka.Cluster.Metrics" = akka-cluster-metrics
    "Akka.Cluster.Metrics.MemoryMetricsSelector, Akka.Cluster.Metrics" = akka-cluster-metrics
  }

  # Globally unique metrics extension serializer identifier.
  serialization-identifiers {
    "Akka.Cluster.Metrics.Serialization.ClusterMetricsMessageSerializer, Akka.Cluster.Metrics" = 10
  }

  #  Provide routing of messages based on cluster metrics.
  router.type-mapping {
    cluster-metrics-adaptive-pool = "Akka.Cluster.Metrics.AdaptiveLoadBalancingPool, Akka.Cluster.Metrics"
    cluster-metrics-adaptive-group = "Akka.Cluster.Metrics.AdaptiveLoadBalancingGroup, Akka.Cluster.Metrics"
  }
}
```