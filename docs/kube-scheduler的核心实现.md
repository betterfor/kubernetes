kube-scheduler组件是kubernetes核心组件之一，主要负责整个集群pod资源对象的调度。

kube-scheduler调度器在为Pod资源对象选择合适节点时，有如下两种最优解：
- 全局最优解：每个调度周期都会遍历kubernetes集群中的所有节点，以便找出全局最优的节点
- 局部最优解：每个调度周期只会遍历部分kubernetes集群中的节点，找出局部最优的节点

全局最优解和局部最优解可以解决调度器在小型和大型kubernetes集群规模上的性能问题。

kube-scheduler组件的主要逻辑在于，如何在kubernetes集群中为一个Pod资源对象找到合适的节点。
调度器每次只调度一个Pod资源对象，为每一个Pod资源对象寻找合适节点的过程就是一个调度周期。

### kube-scheduler组件的启动流程

#### 1、内置调度算法的注册

`NewLegacyRegistry`注册预选和优选算法

#### 2、Cobra命令行参数解析

#### 3、实例化Scheduler对象

[pkg/scheduler/scheduler.go]
```go
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {
```

#### 4、运行EventBoardcaster事件管理器

`cc.EventBroadcaster.StartRecordingToSink(ctx.Done())`注册事件输出到klog

#### 5、运行HTTP或HTTPS服务

- /healthz：用于健康检查
- /metrics：用于监控指标，一般用于Prometheus指标采集
- /debug/pprof：用于pprof性能分析

#### 6、运行Informer同步资源

```go
    // Start all informers.
	cc.InformerFactory.Start(ctx.Done())

	// Wait for all caches to sync before scheduling.
	cc.InformerFactory.WaitForCacheSync(ctx.Done())
```

#### 7、领导者选举实例化

```go
    // If leader election is enabled, runCommand via LeaderElector until done and exit.
	if cc.LeaderElection != nil {
		cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
				close(waitingForLeader)
				sched.Run(ctx)
			},
			OnStoppedLeading: func() {
				klog.Fatalf("leaderelection lost")
			},
		}
		leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}

		leaderElector.Run(ctx)

		return fmt.Errorf("lost lease")
	}
```
有2个回调函数，`OnStartedLeading`函数是当前节点领导者选举成功后回调的函数，该函数定义了kube-scheduler组件的主逻辑；`OnStoppedLeading`函数是当前节点领导者被抢占后回调的函数，在领导者被抢占后，会退出当前的kube-scheduler进程。

#### 8、运行sched.Run调度器

[pkg/scheduler/scheduler.go]
```go
// Run begins watching and scheduling. It starts scheduling and blocked until the context is done.
func (sched *Scheduler) Run(ctx context.Context) {
	sched.SchedulingQueue.Run()
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
	sched.SchedulingQueue.Close()
}
```
`sched.scheduleOne`是kube-scheduler组件的主逻辑，通过wait.Until定时执行，内部会定时调用sched.scheduleOne函数，当sche.config.StopEverything Chan关闭时，该定时器才会停止并退出。

### 优先级与抢占机制

在当前kubernetes系统中，Pod资源对象支持优先级（Priority）与抢占（Preempt）机制。当kube-scheduler调度器运行时，根据Pod资源对象的优先级进行调度，高优先级的Pod资源对象
排在调度队列（SchedulingQueue）的前面，优先获取合适的节点（Node），然后为低优先级的Pod资源对象选择合适的节点。

当高优先级的Pod资源对象没有找到合适的节点时，调度器会尝试抢占低优先级的Pod资源对象的节点，抢占过程是将低优先级的Pod资源对象从所在的节点上驱逐，使高优先级的Pod资源对象运行在该节点上，
被驱逐走的低优先级的Pod资源对象会重新进入调度队列并等待再次选择合适的节点。

### 亲和性调度

[staging/src/k8s.io/api/core/v1/types.go]
```go
// Affinity is a group of affinity scheduling rules.
type Affinity struct {
	// Describes node affinity scheduling rules for the pod.
	// +optional
	NodeAffinity *NodeAffinity `json:"nodeAffinity,omitempty" protobuf:"bytes,1,opt,name=nodeAffinity"`
	// Describes pod affinity scheduling rules (e.g. co-locate this pod in the same node, zone, etc. as some other pod(s)).
	// +optional
	PodAffinity *PodAffinity `json:"podAffinity,omitempty" protobuf:"bytes,2,opt,name=podAffinity"`
	// Describes pod anti-affinity scheduling rules (e.g. avoid putting this pod in the same node, zone, etc. as some other pod(s)).
	// +optional
	PodAntiAffinity *PodAntiAffinity `json:"podAntiAffinity,omitempty" protobuf:"bytes,3,opt,name=podAntiAffinity"`
}
```

### 内置调度算法

`NewLegacyRegistry`定义了调度算法

- 预选调度算法

- 优选调度算法

### 调度器核心实现

