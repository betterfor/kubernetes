## 一、RESTStorage存储服务通用接口

kubernetes的每种资源（包括子资源）都提供了RESTful风格的对外资源存储服务API（即RESTStorage接口），
所有通过RESTful API对外暴露的资源都必须实现RESTStorage接口。

[staging/src/k8s.io/apiserver/pkg/registry/rest/rest.go]
```go
// Storage is a generic interface for RESTful storage services.
// Resources which are exported to the RESTful API of apiserver need to implement this interface. It is expected
// that objects may implement any of the below interfaces.
type Storage interface {
	// New returns an empty object that can be used with Create and Update after request data has been put into it.
	// This object must be a pointer type for use with Codec.DecodeInto([]byte, runtime.Object)
	New() runtime.Object
}
```
kubernetes的每种资源实现的RESTStorage接口一般定义在`pkg/registry/<资源组>/<资源>/storage/storage.go`中，它们通过`NewStorage`函数或`NewREST`函数实例化。

[pkg/registry/apps/deployment/storage/storage.go]
```go
// DeploymentStorage includes dummy storage for Deployments and for Scale subresource.
type DeploymentStorage struct {
	Deployment *REST
	Status     *StatusREST
	Scale      *ScaleREST
	Rollback   *RollbackREST
}
// REST implements a RESTStorage for Deployments.
type REST struct {
	*genericregistry.Store
	categories []string
}
// StatusREST implements the REST endpoint for changing the status of a deployment
type StatusREST struct {
	store *genericregistry.Store
}
```
Deployment资源定义了REST数据结构与StatusREST数据结构，其中REST数据结构用于实现Deployment资源的RESTStorage接口，而StatusREST数据结构用于实现deployment/status子资源的RESTStorage接口。
每个RESTStorage接口都对RegistryStore操作进行了封装。
例如，对deployment/status子资源进行GET操作时，实际执行的是RegistryStore操作。
```go
// Get retrieves the object from the storage. It is required to support Patch.
func (r *StatusREST) Get(ctx context.Context, name string, options *metav1.GetOptions) (runtime.Object, error) {
	return r.store.Get(ctx, name, options)
}
```

## 二、RegistryStore存储服务通用操作

RegistryStore实现了资源存储的通用操作，例如，在存储资源对象之前执行某个函数，在存储对象资源执行某个函数。

- Before Func：预处理，定义在创建资源对象之前调用，做一些预处理的工作
- After Func：定义为在创建资源对象之后调用，做一些收尾工作

[staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go]
```go
type Store struct {
	CreateStrategy rest.RESTCreateStrategy
	AfterCreate ObjectFunc
	UpdateStrategy rest.RESTUpdateStrategy
	AfterUpdate ObjectFunc
	DeleteStrategy rest.RESTDeleteStrategy
	AfterDelete ObjectFunc
	ReturnDeletedObject bool
	ShouldDeleteDuringUpdate func(ctx context.Context, key string, obj, existing runtime.Object) bool
	ExportStrategy rest.RESTExportStrategy
	TableConvertor rest.TableConvertor
	Storage DryRunnableStorage
    ...
}
```
有4种Strategy预处理方法，分别是`CreateStrategy`(创建资源对象时的预处理操作)、`UpdateStrategy`(更新资源对象时的预处理操作)、`DeleteStrategy`(删除资源对象时的预处理操作)、`ExportStrategy`(导出资源对象时的预处理操作)。

定义了3种创建资源对象后的处理方法，分别是`AfterCreate`(创建资源对象后的处理操作)、`AfterUpdate`(更新资源对象后的处理操作)、`AfterDelete`(删除资源对象后的处理操作)。

Storage字段是RegistryStore对Storage.Interface通用存储接口进行的封装，实现了对ETCD集群的读/写操作。

```go
func (e *Store) Create(ctx context.Context, obj runtime.Object, createValidation rest.ValidateObjectFunc, options *metav1.CreateOptions) (runtime.Object, error) {
    // 1、通过rest.BeforeCreate函数执行预处理操作
	if err := rest.BeforeCreate(e.CreateStrategy, ctx, obj); err != nil {
		return nil, err
	}
	...
	out := e.NewFunc()
    // 2、通过e.Storage.Create函数创建资源对象
	if err := e.Storage.Create(ctx, key, obj, out, ttl, dryrun.IsDryRun(options.DryRun)); err != nil {
		err = storeerr.InterpretCreateError(err, qualifiedResource, name)
		err = rest.CheckGeneratedNameError(e.CreateStrategy, err, obj)
		if !apierrors.IsAlreadyExists(err) {
			return nil, err
		}
		if errGet := e.Storage.Get(ctx, key, storage.GetOptions{}, out); errGet != nil {
			return nil, err
		}
		accessor, errGetAcc := meta.Accessor(out)
		if errGetAcc != nil {
			return nil, err
		}
		if accessor.GetDeletionTimestamp() != nil {
			msg := &err.(*apierrors.StatusError).ErrStatus.Message
			*msg = fmt.Sprintf("object is being deleted: %s", *msg)
		}
		return nil, err
	}
    // 3、通过e.AfterCreate函数执行收尾操作
	if e.AfterCreate != nil {
		if err := e.AfterCreate(out); err != nil {
			return nil, err
		}
	}
	if e.Decorator != nil {
		if err := e.Decorator(out); err != nil {
			return nil, err
		}
	}
	return out, nil
}
```

## 三、Storage.Interface通用存储接口

[staging/src/k8s.io/apiserver/pkg/storage/interfaces.go]
```go
type Interface interface {
	// 资源版本管理器，用于管理ETCD集群中的数据版本对象
	Versioner() Versioner
	// 创建资源对象的方法
	Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error
	// 删除资源对象的方法
	Delete(ctx context.Context, key string, out runtime.Object, preconditions *Preconditions, validateDeletion ValidateObjectFunc) error
	// 通过Watch机制监控资源对象变化方法，只应用与单个key
	Watch(ctx context.Context, key string, opts ListOptions) (watch.Interface, error)
	// 通过Watch机制监控资源对象变化方法，应用于多个key（当前目录及目录下所有的key）
	WatchList(ctx context.Context, key string, opts ListOptions) (watch.Interface, error)
	// 获取资源对象的方法
	Get(ctx context.Context, key string, opts GetOptions, objPtr runtime.Object) error
	// 获取资源对象的方法，以列表(List)的形式返回
	GetToList(ctx context.Context, key string, opts ListOptions, listObj runtime.Object) error
	// 获取资源对象的方法，以列表(List)的形式返回
	List(ctx context.Context, key string, opts ListOptions, listObj runtime.Object) error
	// 保证传入的tryUpdate函数运行成功
	GuaranteedUpdate(
		ctx context.Context, key string, ptrToType runtime.Object, ignoreNotFound bool,
		precondtions *Preconditions, tryUpdate UpdateFunc, suggestion runtime.Object) error
	// 获取指定key下的条目数量
	Count(key string) (int64, error)
}
```
storage.Interface是通用存储接口，实现通用存储接口的分别是
- Cacher：带有缓存功能的资源存储对象，[staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go]
- store：底层存储对象，真正与etcd集群交互的资源存储对象，[staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go]

[staging/src/k8s.io/apiserver/pkg/server/options/etcd.go]
```go
func (f *SimpleRestOptionsFactory) GetRESTOptions(resource schema.GroupResource) (generic.RESTOptions, error) {
	ret := generic.RESTOptions{
		StorageConfig:           &f.Options.StorageConfig,
		Decorator:               generic.UndecoratedStorage,
		EnableGarbageCollection: f.Options.EnableGarbageCollection,
		DeleteCollectionWorkers: f.Options.DeleteCollectionWorkers,
		ResourcePrefix:          resource.Group + "/" + resource.Resource,
		CountMetricPollPeriod:   f.Options.StorageConfig.CountMetricPollPeriod,
	}
	if f.Options.EnableWatchCache {
		sizes, err := ParseWatchCacheSizes(f.Options.WatchCacheSizes)
		if err != nil {
			return generic.RESTOptions{}, err
		}
		size, ok := sizes[resource]
		if ok && size > 0 {
			klog.Warningf("Dropping watch-cache-size for %v - watchCache size is now dynamic", resource)
		}
		if ok && size <= 0 {
			ret.Decorator = generic.UndecoratedStorage
		} else {
			ret.Decorator = genericregistry.StorageWithCacher()
		}
	}
	return ret, nil
}
```
如果不启用WatchCache功能，kubernetes API Server通过`generic.UndecoratedStorage`函数直接创建store底层存储对象并返回。
在默认情况下，kubernetes API Server缓存功能是开启的，可通过`--watch-cache`参数设置，如果该参数为`true`，则通过`genericregistry.StorageWithCacher()`函数创建带有缓存功能的资源存储对象。

Cacher实际上是在store之上封装了一层缓存层，在初始化的过程中，也会创建store底层存储对象。

> Cacher的初始化过程是基于装饰器模式
```go
type StringManipulator func(string) string
func ToLower(m StringManipulator) StringManipulator {
    return func(s string) string {
        lower := strings.ToLower(s)
        return m(lower)
    }
}
```
首先装饰器基本上是一个函数，它将特定类型的函数作为参数，并返回相同类型的函数。ToLower函数是一个装饰器函数，ToLower的函数返回值与其内部函数返回值是相同类型。

在Cacher实例化过程中，就在装饰器函数里实现了store和Cacher的实例化过程。

[staging/src/k8s.io/apiserver/pkg/registry/generic/registry/storage_factory.go]
```go
// Creates a cacher based given storageConfig.
func StorageWithCacher() generic.StorageDecorator {
	return func(
		storageConfig *storagebackend.Config,
		resourcePrefix string,
		keyFunc func(obj runtime.Object) (string, error),
		newFunc func() runtime.Object,
		newListFunc func() runtime.Object,
		getAttrsFunc storage.AttrFunc,
		triggerFuncs storage.IndexerFuncs,
		indexers *cache.Indexers) (storage.Interface, factory.DestroyFunc, error) {

		s, d, err := generic.NewRawStorage(storageConfig, newFunc)
		if err != nil {
			return s, d, err
		}
		if klog.V(5).Enabled() {
			klog.Infof("Storage caching is enabled for %s", objectTypeToString(newFunc()))
		}

		cacherConfig := cacherstorage.Config{
			Storage:        s,
			Versioner:      etcd3.APIObjectVersioner{},
			ResourcePrefix: resourcePrefix,
			KeyFunc:        keyFunc,
			NewFunc:        newFunc,
			NewListFunc:    newListFunc,
			GetAttrsFunc:   getAttrsFunc,
			IndexerFuncs:   triggerFuncs,
			Indexers:       indexers,
			Codec:          storageConfig.Codec,
		}
		cacher, err := cacherstorage.NewCacherFromConfig(cacherConfig)
		if err != nil {
			return nil, func() {}, err
		}
		destroyFunc := func() {
			cacher.Stop()
			d()
		}

		// TODO : Remove RegisterStorageCleanup below when PR
		// https://github.com/kubernetes/kubernetes/pull/50690
		// merges as that shuts down storage properly
		RegisterStorageCleanup(destroyFunc)

		return cacher, destroyFunc, nil
	}
}
```

## 四、CacherStorage缓存层

缓存（Cache）的应用场景非常广泛，可以使用缓存来降低数据库服务器的负载、减少连接数。

- 缓存命中：客户端发起请求，请求中的数据存于缓存层中，则从缓存层直接返回数据
- 缓存回源：客户端发起请求，请求中的数据未存于缓存层中，此时缓存层向DB数据层获取数据（回源），DB数据层将数据返回给缓存层，缓存层收到数据并将数据更新到自身缓存中，最后将数据返回给客户端。

### 1、CacherStorage缓存层设计

缓存层并非会为所有操作都缓存数据，对于Create、Delete、Count等操作，为了保持数据一致性，没有必要再封装一层缓存层，直接通过store向etcd发起请求。
只有Get、GetToList、List、GuaranteedUpdate、Watch、WatchList等操作是基于缓存设计的。

#### 1、cacherWatcher

[staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go]
```go
func (c *Cacher) Watch(ctx context.Context, key string, opts storage.ListOptions) (watch.Interface, error) {
	// Create a watcher here to reduce memory allocations under lock,
	// given that memory allocation may trigger GC and block the thread.
	// Also note that emptyFunc is a placeholder, until we will be able
	// to compute watcher.forget function (which has to happen under lock).
	watcher := newCacheWatcher(chanSize, filterWithAttrsFunction(key, pred), emptyFunc, c.versioner, deadline, pred.AllowWatchBookmarks, c.objectType)
	c.watchCache.RLock()
	defer c.watchCache.RUnlock()
	initEvents, err := c.watchCache.GetAllEventsSinceThreadUnsafe(watchRV)
	if err != nil {
		// To match the uncached watch implementation, once we have passed authn/authz/admission,
		// and successfully parsed a resource version, other errors must fail with a watch event of type ERROR,
		// rather than a directly returned error.
		return newErrWatcher(err), nil
	}

	// With some events already sent, update resourceVersion so that
	// events that were buffered and not yet processed won't be delivered
	// to this watcher second time causing going back in time.
	if len(initEvents) > 0 {
		watchRV = initEvents[len(initEvents)-1].ResourceVersion
	}

	func() {
		c.Lock()
		defer c.Unlock()
		// Update watcher.forget function once we can compute it.
		watcher.forget = forgetWatcher(c, c.watcherIdx, triggerValue, triggerSupported)
		c.watchers.addWatcher(watcher, c.watcherIdx, triggerValue, triggerSupported)

		// Add it to the queue only when the client support watch bookmarks.
		if watcher.allowWatchBookmarks {
			c.bookmarkWatchers.addWatcher(watcher)
		}
		c.watcherIdx++
	}()

	go watcher.process(ctx, initEvents, watchRV)
	return watcher, nil
}
```
每一个发起Watch请求的客户端都会分配一个cacheWatcher，用于客户端接受Watch事件。

当客户端发起Watch请求时，通过`newCacheWatcher`函数实例化cacheWatch对象，并为其分配一个唯一id，从0开始计数，每次有新的客户端发送Watch请求时，该id会自增1.但在Kubernetes API Server重启时清零。
最后，将对象添加到c.watchers进行统一管理。最后执行Add(添加)、Delete(删除)、Terminate(中止)等操作

cacheWatch通过map数据结构进行管理，其中key为id，value为cacheWatcher。
```go
type watchersMap map[int]*cacheWatcher
```

在`newCacheWatcher`初始化后，会运行`process`监控`c.input`中的数据，当其中没有数据时，阻塞；当有数据时，会发送大于`resourceVersion`资源版本号的数据。

```go
func (c *cacheWatcher) process(ctx context.Context, initEvents []*watchCacheEvent, resourceVersion uint64) {
	...
	for {
		select {
		case event, ok := <-c.input:
			if !ok {
				return
			}
			// only send events newer than resourceVersion
			if event.ResourceVersion > resourceVersion {
				c.sendWatchCacheEvent(event)
			}
		case <-ctx.Done():
			return
		}
	}
}
``` 

#### 2、watchCache

在实例化过程中会使用Reflector框架的ListAndWatch函数通过store监控etcd集群的Watch事件

watchCache接收Reflector框架的事件回调，并实现了Add、Update、Delete方法，分别用于接收watch.Added、watch.Modified、watch.Deleted事件。

- eventHandler：将事件回调给Cacher，Cacher分发给目前所有的连接者
- w.cache：将事件存储至滑动窗口，它提供了对Watch操作的缓存数据，防止因网络或其他原因观察者连接中断导致的事件丢失
- cache.Store：将事件存储至本地缓存，cache.Store与client-go下Indexer功能相同

[staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go]
```go
func (w *watchCache) processEvent(event watch.Event, resourceVersion uint64, updateFunc func(*storeElement) error) error {
	key, err := w.keyFunc(event.Object)
	if err != nil {
		return fmt.Errorf("couldn't compute key: %v", err)
	}
	elem := &storeElement{Key: key, Object: event.Object}
	elem.Labels, elem.Fields, err = w.getAttrsFunc(event.Object)
	if err != nil {
		return err
	}

	wcEvent := &watchCacheEvent{
		Type:            event.Type,
		Object:          elem.Object,
		ObjLabels:       elem.Labels,
		ObjFields:       elem.Fields,
		Key:             key,
		ResourceVersion: resourceVersion,
		RecordTime:      w.clock.Now(),
	}

	if err := func() error {
		// TODO: We should consider moving this lock below after the watchCacheEvent
		// is created. In such situation, the only problematic scenario is Replace(
		// happening after getting object from store and before acquiring a lock.
		// Maybe introduce another lock for this purpose.
		w.Lock()
		defer w.Unlock()

		previous, exists, err := w.store.Get(elem)
		if err != nil {
			return err
		}
		if exists {
			previousElem := previous.(*storeElement)
			wcEvent.PrevObject = previousElem.Object
			wcEvent.PrevObjLabels = previousElem.Labels
			wcEvent.PrevObjFields = previousElem.Fields
		}

		w.updateCache(wcEvent)
		w.resourceVersion = resourceVersion
		defer w.cond.Broadcast()

		return updateFunc(elem)
	}(); err != nil {
		return err
	}

	// Avoid calling event handler under lock.
	// This is safe as long as there is at most one call to Add/Update/Delete and
	// UpdateResourceVersion in flight at any point in time, which is true now,
	// because reflector calls them synchronously from its main thread.
	if w.eventHandler != nil {
		w.eventHandler(wcEvent)
	}
	return nil
}
```

eventHandler回调Cacher

[staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go]
```go
func (c *Cacher) processEvent(event *watchCacheEvent) {
	if curLen := int64(len(c.incoming)); c.incomingHWM.Update(curLen) {
		// Monitor if this gets backed up, and how much.
		klog.V(1).Infof("cacher (%v): %v objects queued in incoming channel.", c.objectType.String(), curLen)
	}
	c.incoming <- *event
}
```

#### 3、Cacher

Cacher接收到watchCache回调的事件，遍历所有已连接的观察者，并将事件逐个分发给每个观察者，该过程通过非阻塞机制实现，不会阻塞任何一个观察者。
dispatchEvents->dispatchEvent->watcher.add

[staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go]
```go
func (c *cacheWatcher) add(event *watchCacheEvent, timer *time.Timer) bool {
	// Try to send the event immediately, without blocking.
	if c.nonblockingAdd(event) {
		return true
	}

	closeFunc := func() {
		// This means that we couldn't send event to that watcher.
		// Since we don't want to block on it infinitely,
		// we simply terminate it.
		klog.V(1).Infof("Forcing watcher close due to unresponsiveness: %v", c.objectType.String())
		c.forget()
	}

	if timer == nil {
		closeFunc()
		return false
	}

	// OK, block sending, but only until timer fires.
	select {
	case c.input <- event:
		return true
	case <-timer.C:
		closeFunc()
		return false
	}
}
```

### 2、ResourceVersion资源版本号

所有的kubernetes资源都有一个资源版本号（Resource Version），其用于表示资源存储版本，一般定义在元数据中。每次在修改ETCD集群存储的资源对象时，kubernetes API Server都会更改ResourceVersion，
使得client-go执行Watch操作时可以根据ResourceVersion来确定资源对象是否发生变化。当client-go断开时，只要从上一次的ResourceVersion继续监控（Watch操作），就能获取历史事件，这样可以防止事件丢失。

kubernetes API Server依赖于ETCD集群中的全局Index机制来进行管理。在ETCD集群中，有两个比较关键的Index，用于跟踪Etcd集群中的数据

- createdIndex：全局唯一递增的正整数。每次在Etcd集群中创建key时会递增
- modifiedIndex：每次在Etcd集群中修改key时会递增

两个index都是原子操作，其中modifiedIndex机制被kubernetes系统用于获取资源版本号。
kubernetes系统通过资源版本号来实现乐观并发控制（乐观锁）。
当处理多客户端并发的事务时，事务之间不会互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。
在提交更新数据前，各个事务都会先检查在自己读取数据之后，有没有其他事务又修改了该数据。如果有其他事务更新过数据，那么正在提交数据的事务会进行回滚。

### 3、watchCache滑动窗口

常用的缓存算法

1、FIFO（First Input First Output）
- 特点：先进先出
- 数据结构：队列
- 淘汰原则：当缓存满时，将最先进入缓存的数据淘汰

2、LRU（Least Recently Used）
- 特点：最近最少使用，优先移除最久未使用的数据，按时间维度衡量
- 数据结构：链表和HashMap
- 淘汰原则：根据缓存数据使用的时间，将最不经常使用的缓存数据优先淘汰。如果一个数据在最近一段时间都没有被访问，那么在将来它被访问的可能性也很小。

3、LFU（Least Frequently Used）
- 特点：最近最不常使用，优先移除访问次数最少的数据，按统计维度衡量
- 数据结构：数组、HashMap和堆
- 淘汰原则：根据缓存数据的使用次数，将访问次数最少的缓存数据优先淘汰。如果一个数据在最近一段时间内使用次数很少，那么在将来被使用的可能性也很小。

[staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go]
```go
type watchCache struct {
	// 滑动窗口的大小
	capacity int

	// upper bound of capacity since event cache has a dynamic size.
	upperBoundCapacity int

	// lower bound of capacity since event cache has a dynamic size.
	lowerBoundCapacity int

	// keyFunc is used to get a key in the underlying storage for a given object.
	keyFunc func(runtime.Object) (string, error)

	// getAttrsFunc is used to get labels and fields of an object.
	getAttrsFunc func(runtime.Object) (labels.Set, fields.Set, error)

	// 缓存滑动窗口，可以通过一个固定大小的数组向前滑动。当滑动窗口满的时候，将最先进入缓存滑动窗口的数据淘汰
	cache      []*watchCacheEvent
	startIndex int // 开始下标
	endIndex   int // 结束下标
    ...
}
```
假设缓存滑动初始固定大小为3，startIndex=endIndex=0，接受到事件后，将事件放入滑动窗口头部，当滑动窗口满时，取模运算覆盖数据
```go
// Assumes that lock is already held for write.
func (w *watchCache) updateCache(event *watchCacheEvent) {
	w.resizeCacheLocked(event.RecordTime)
	if w.isCacheFullLocked() {
		// Cache is full - remove the oldest element.
		w.startIndex++
	}
	w.cache[w.endIndex%w.capacity] = event
	w.endIndex++
}
```
断点续传功能
```go
func (w *watchCache) GetAllEventsSinceThreadUnsafe(resourceVersion uint64) ([]*watchCacheEvent, error) {
	size := w.endIndex - w.startIndex
	var oldest uint64
	switch {
	case w.listResourceVersion > 0 && w.startIndex == 0:
		// If no event was removed from the buffer since last relist, the oldest watch
		// event we can deliver is one greater than the resource version of the list.
		oldest = w.listResourceVersion + 1
	case size > 0:
		// If the previous condition is not satisfied: either some event was already
		// removed from the buffer or we've never completed a list (the latter can
		// only happen in unit tests that populate the buffer without performing
		// list/replace operations), the oldest watch event we can deliver is the first
		// one in the buffer.
		oldest = w.cache[w.startIndex%w.capacity].ResourceVersion
	default:
		return nil, fmt.Errorf("watch cache isn't correctly initialized")
	}

	if resourceVersion == 0 {
		// resourceVersion = 0 从本地缓存获取历史事件并返回
		allItems := w.store.List()
		result := make([]*watchCacheEvent, len(allItems))
		for i, item := range allItems {
			elem, ok := item.(*storeElement)
			if !ok {
				return nil, fmt.Errorf("not a storeElement: %v", elem)
			}
			objLabels, objFields, err := w.getAttrsFunc(elem.Object)
			if err != nil {
				return nil, err
			}
			result[i] = &watchCacheEvent{
				Type:            watch.Added,
				Object:          elem.Object,
				ObjLabels:       objLabels,
				ObjFields:       objFields,
				Key:             elem.Key,
				ResourceVersion: w.resourceVersion,
			}
		}
		return result, nil
	}
	if resourceVersion < oldest-1 {
		return nil, errors.NewResourceExpired(fmt.Sprintf("too old resource version: %d (%d)", resourceVersion, oldest-1))
	}

	// Binary search the smallest index at which resourceVersion is greater than the given one.
	f := func(i int) bool {
		return w.cache[(w.startIndex+i)%w.capacity].ResourceVersion > resourceVersion
	}
	first := sort.Search(size, f)
	result := make([]*watchCacheEvent, size-first)
	for i := 0; i < size-first; i++ {
		result[i] = w.cache[(w.startIndex+first+i)%w.capacity]
	}
	return result, nil
}
```


## 五、UnderlyingStorage底层存储对象

[staging/src/k8s.io/apiserver/pkg/storage/storagebackend/factory/factory.go]
```go
// Create creates a storage backend based on given config.
func Create(c storagebackend.Config, newFunc func() runtime.Object) (storage.Interface, DestroyFunc, error) {
	switch c.Type {
	case "etcd2":
		return nil, nil, fmt.Errorf("%v is no longer a supported storage backend", c.Type)
	case storagebackend.StorageTypeUnset, storagebackend.StorageTypeETCD3:
		return newETCD3Storage(c, newFunc)
	default:
		return nil, nil, fmt.Errorf("unknown storage type: %s", c.Type)
	}
}
```
资源对象在Etcd集群中以二进制形式存储（即application/vnd.kubernetes.protobuf），存储和获取过程都通过protobufSerializer编解码进行数据的编码和解码。

[staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go]
```go
// Get implements storage.Interface.Get.
func (s *store) Get(ctx context.Context, key string, opts storage.GetOptions, out runtime.Object) error {
	key = path.Join(s.pathPrefix, key)
	startTime := time.Now()
	getResp, err := s.client.KV.Get(ctx, key)
	metrics.RecordEtcdRequestLatency("get", getTypeName(out), startTime)
    ...
	return decode(s.codec, s.versioner, data, out, kv.ModRevision)
}
func decode(codec runtime.Codec, versioner storage.Versioner, value []byte, objPtr runtime.Object, rev int64) error {
	if _, err := conversion.EnforcePtr(objPtr); err != nil {
		return fmt.Errorf("unable to convert output object to pointer: %v", err)
	}
	_, _, err := codec.Decode(value, nil, objPtr)
	if err != nil {
		return err
	}
	// being unable to set the version does not prevent the object from being extracted
	if err := versioner.UpdateObject(objPtr, uint64(rev)); err != nil {
		klog.Errorf("failed to update object version: %v", err)
	}
	return nil
}
```
Get流程：

1、通过s.client.KV.Get获取Etcd集群中Pod资源对象的数据

2、通过protobufSerializer编解码（即codec.Decode函数）解码二进制，解码后的数据存放在objPtr中

3、最后通过versioner.UpdateObject函数更新最新的资源对象ResourceVersion资源版本号

## 六、Codec编解码数据

Example
```go
package main

import (
	"context"
	"fmt"
	"go.etcd.io/etcd/clientv3"
	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/runtime/serializer"
	"time"
)

var Scheme = runtime.NewScheme()
var Codecs = serializer.NewCodecFactory(Scheme)
var inMediaType = "application/vnd.kubernetes.protobuf"
var outMediaType = "application/json"

func init() {
	v1.AddToScheme(Scheme)
}

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"localhost:2379", "localhost:22379", "localhost:32379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		panic(err)
	}
	defer cli.Close()

	resp, err := cli.Get(context.Background(), "/registry/pods/default/centos-xxx")
	if err != nil {
		panic(err)
	}
	kv := resp.Kvs[0]

	inCodec := newCodec(inMediaType)
	outMediaType := newCodec(inMediaType)

	obj, err := runtime.Decode(inCodec, kv.Value)
	if err != nil {
		panic(err)
	}
	fmt.Println("Decode ---")
	fmt.Println(obj)

	encoded, err := runtime.Encode(outMediaType, obj)
	if err != nil {
		panic(err)
	}
	fmt.Println("Encode ---")
	fmt.Println(string(encoded))
}

func newCodec(mediaTypes string) runtime.Codec {
	info, ok := runtime.SerializerInfoForMediaType(Codecs.SupportedMediaTypes(), mediaTypes)
	if !ok {
		panic(fmt.Errorf("no serializers registred for %v", mediaTypes))
	}
	cfactory := serializer.DirectCodecFactory{CodecFactory: Codecs}

	gv, err := schema.ParseGroupVersion("v1")
	if err != nil {
		panic("unexpected error")
	}
	encoder := cfactory.EncoderForVersion(info.Serializer, gv)
	decoder := cfactory.DecoderToVersion(info.Serializer, gv)
	return cfactory.CodecForVersions(encoder, decoder, gv, gv)
}
```
1、实例化Scheme资源注册表及Codecs编解码器，并通过init函数将corev1资源组下的资源注册至Scheme资源注册表下中，这是因为要对Pod资源数据进行解码操作。
inMediaType定义了编码格式，outMediaType定义了解码格式。

2、通过clientv3.New函数实例化Etcd Client对象，并设置参数，例如Endpoints连接Etcd集群的地址，设置超时时间，通过cli.Get获取pod资源对象数据

3、通过newCodec函数实例化runtime.Codec编解码器，分别实例化inCodec编码器对象、outCodec解码器对象

4、通过runtime.Decode解码器（即protobufSerializer）解码资源对象数据并输出

5、通过runtime.Encode编码器（即jsonSerializer）编码资源对象并输出

### 七、Strategy预处理

在kubernetes系统中，每种资源都有自己的预处理（Strategy）操作，其用于创建、更新、删除、导出资源对象前对资源执行预处理操作，例如在存储资源对象之前验证或修改对象。
每个资源的特殊需求都可以在自己的Strategy预处理接口中实现。每个资源的Strategy预处理接口代码一般定义在pkg/registry/<资源组>/<资源>/strategy.go

**Strategy预处理接口定义**

[staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go]
```go
type GenericStore interface {
	GetCreateStrategy() rest.RESTCreateStrategy	// 创建资源对象时的预处理操作
	GetUpdateStrategy() rest.RESTUpdateStrategy // 更新资源对象时的预处理操作
	GetDeleteStrategy() rest.RESTDeleteStrategy // 删除资源对象时的预处理操作
	GetExportStrategy() rest.RESTExportStrategy // 导出资源对象时的预处理操作
}
```
#### 1、创建资源对象时的预处理操作

[staging/src/k8s.io/apiserver/pkg/registry/rest/create.go]
```go
type RESTCreateStrategy interface {
	// 判断当前资源是否拥有所属的命名空间，如有所有的命名空间，则返回true
	NamespaceScoped() bool
	// 创建当前资源对象之前的处理函数
	PrepareForCreate(ctx context.Context, obj runtime.Object)
	// 创建当前资源对象之前的验证函数。验证资源对象的字段信息，此方法不会修改对象
	Validate(ctx context.Context, obj runtime.Object) field.ErrorList
	// 在创建当前资源对象之前将存储的资源对象规范化。在当前kubernetes未使用该方法
	Canonicalize(obj runtime.Object)
}
```

CreateStrategy提供了BeforeCreate函数，对CreateStrategy操作的方法进行了打包、封装，并使各方法顺序进行。

#### 2、更新资源对象时的预处理操作

[staging/src/k8s.io/apiserver/pkg/registry/rest/update.go]
```go
type RESTUpdateStrategy interface {
	// 判断当前资源是否拥有所属的命名空间，如有所有的命名空间，则返回true
	NamespaceScoped() bool
	// 在更新当前资源对象时，如果资源对象已经存在，确定是否允许重新创建资源对象
	AllowCreateOnUpdate() bool
	// 更新当前对象之前的处理函数
	PrepareForUpdate(ctx context.Context, obj, old runtime.Object)
	// 更新当前资源对象之前的验证函数。验证资源对象的字段信息，此方法不会修改对象
	ValidateUpdate(ctx context.Context, obj, old runtime.Object) field.ErrorList
	// 在创建当前资源对象之前将存储的资源对象规范化。在当前kubernetes未使用该方法
	Canonicalize(obj runtime.Object)
	// 在更新当前资源对象时，如果未指定资源版本，确定是否运行更新操作
	AllowUnconditionalUpdate() bool
}
```
UpdateStrategy提供了BeforeUpdate函数

#### 3、删除资源对象时的预处理操作

[staging/src/k8s.io/apiserver/pkg/registry/rest/delete.go]
```go
type RESTDeleteStrategy interface {
	runtime.ObjectTyper
}
type RESTGracefulDeleteStrategy interface {
	// CheckGracefulDelete should return true if the object can be gracefully deleted and set
	// any default values on the DeleteOptions.
	CheckGracefulDelete(ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) bool
}
```
RESTDeleteStrategy接口没有定义任何预定义方法，但可以显式转换RESTGracefulDeleteStrategy接口，该接口的CheckGracefulDelete方法用于检查资源对象是否支持优雅删除。

DeleteStrategy提供BeforeDelete函数

```go
func BeforeDelete(strategy RESTDeleteStrategy, ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) (graceful, gracefulPending bool, err error) {
	...
	gracefulStrategy, ok := strategy.(RESTGracefulDeleteStrategy)
	...
	if !gracefulStrategy.CheckGracefulDelete(ctx, obj, options) {
		return false, false, nil
	}
	...
	return true, false, nil
}
```

#### 4、导出资源对象时的预处理操作

[staging/src/k8s.io/apiserver/pkg/registry/rest/export.go]
```go
type RESTExportStrategy interface {
	// Export strips fields that can not be set by the user.  If 'exact' is false
	// fields specific to the cluster are also stripped
	Export(ctx context.Context, obj runtime.Object, exact bool) error
}
```
ExportStrategy操作之定义了Export方法，只有部分资源实现了该方法，例如Service。在导出Service资源对象时，对该资源对象的Spec.ClusterIP、Spec.Type进行了判断和修改

[pkg/registry/core/service/strategy.go]
```go
func (svcStrategy) Export(ctx context.Context, obj runtime.Object, exact bool) error {
	t, ok := obj.(*api.Service)
	if !ok {
		// unexpected programmer error
		return fmt.Errorf("unexpected object: %v", obj)
	}
	// TODO: service does not yet have a prepare create strategy (see above)
	t.Status = api.ServiceStatus{}
	if exact {
		return nil
	}
	//set ClusterIPs as nil - if ClusterIPs[0] != None
	if len(t.Spec.ClusterIPs) > 0 && t.Spec.ClusterIPs[0] != api.ClusterIPNone {
		t.Spec.ClusterIP = ""
		t.Spec.ClusterIPs = nil
	}
	if t.Spec.Type == api.ServiceTypeNodePort {
		for i := range t.Spec.Ports {
			t.Spec.Ports[i].NodePort = 0
		}
	}
	return nil
}
```