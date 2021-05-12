## 源码目录

| 源码目录   | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| discovery  | 提供DiscoveryClient发现客户端                                |
| dynamic    | 提供DynamicClient动态客户端                                  |
| informers  | 每种Kubernetes资源的Informer实现                             |
| kubernetes | 提供ClientSet客户端                                          |
| listers    | 为每一个kubernetes资源提供Lister功能，该功能对Get和List请求提供只读的缓存数据 |
| plugin     | 提供OpenStack、GCP和Azure等云服务商授权插件                  |
| rest       | 提供RESTClient客户端，对Kubernetes API Server执行RESTful操作 |
| scale      | 提供ScaleClient客户端，用于扩容或缩容Deployment、ReplicaSet、RC等资源对象 |
| tools      | 提供常用工具，例如SharedInformer、Reflector、DealtFIFO及Indexers。提供Client查询和缓存机制，以减少向kube-apiserver发起的请求数等 |
| transport  | 提供安全的TCP连接，支持Http Stream，某些操作需要在客户端和容器之间的传输二进制流，例如exec、attach等操作。该功能由内部的spdy包提供支持。 |
| util       | 提供常用方法，例如WorkQueue工作队列、Certificate证书管理等。 |

RESTClient是最基础的客户端。RESTClient对HTTP Request进行了封装，实现了RESTful风格的API。ClientSet、DynamicClient及DiscoveryClient客户端都是基于RESTClient实现的。

- ClientSet在RESTClient的基础上封装了对Resource和Version的管理方法。每个Resource可以理解为一个客户端，而ClientSet则是多个客户端的集合，每个Resource和Version都以函数的形式暴露给开发者。而ClientSet只能够处理kubernetes内置资源，是由client-gen代码生成器自动生成的。
- DynamicClient与ClientSet最大的不同之处是，ClientSet仅能访问kubernetes内置资源，不能直接访问CRD自定义资源。DynamicClient能够处理kubernetes中的所有资源对象。
- DiscoveryClient发现客户端，用于发现kube-apiserver所支持的资源组、资源版本，资源信息（Group、Version、Resources）

### 1、kubeconfig配置管理

kubeconfig用于管理访问kube-apiserver的配置信息，同时也支持访问多kube-apiserver的配置管理，可以在不同的环境下管理不同的kube-apiserver集群配置，不同的业务线也可以拥有不同的集群。kubernetes的其他组件都使用kubeconfig配置信息来连接kube-apiserver组件，例如当kubectl访问kube-apiserver时，会默认加载kubeconfig配置信息。

kubeconfig中存储了集群、用户、命名空间和身份验证等信息，在默认情况下，kubeconfig会存放在`$HOME/.kube/config`路径下。kubeconfig配置信息如下：

```bash
$ cat $HOME/.kube/config
apiVesion: v1
kind: Config
preferences: {}

clusters:
- cluster:
  name: dev-cluster
users:
- name: dev-user
contexts:
- context
  name: dev-context
```

kubeconfig配置信息通常包含3个部分，如下：

- clusters：定义kubernetes集群信息，例如kube-apiserver的服务地址及集群的证书信息等。
- users：定义kubernetes集群用户身份验证的客户端凭据，例如client-certificate,client-key,token及username/password等。
- contexts：定义kubernetes集群用户信息和命名空间等，用于将请求发送到指定的集群。

client-go会读取kubeconfig配置信息并生成config对象。

```go
package main
import "k8s.io/client-go/tools/clientcmd"

func main() {
    config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
    ...
}
```

`clientcmd.BuildConfigFromFlags`函数会读取kubeconfig配置信息并实例化rest.Config对象。其中kubeconfig最核心的功能是管理多个访问kube-apiserver集群的配置信息，将多个配置信息合并(merge)成一份，在合并的过程中会解决多个配置文件字段冲突的问题。该过程由Load函数完成，可分为两步：1、加载kubeconfig配置信息；2、合并多个kubeconfig配置信息。

[staging/src/k8s.io/client-go/tools/clientcmd/loader.go]

```go
func (rules *ClientConfigLoadingRules) Load() (*clientcmdapi.Config, error) {
	...
	kubeConfigFiles := []string{}

	// Make sure a file we were explicitly told to use exists
	if len(rules.ExplicitPath) > 0 {
		if _, err := os.Stat(rules.ExplicitPath); os.IsNotExist(err) {
			return nil, err
		}
		kubeConfigFiles = append(kubeConfigFiles, rules.ExplicitPath)

	} else {
		kubeConfigFiles = append(kubeConfigFiles, rules.Precedence...)
	}

	kubeconfigs := []*clientcmdapi.Config{}
	// read and cache the config files so that we only look at them once
	for _, filename := range kubeConfigFiles {
		...
		config, err := LoadFromFile(filename)
         ...
		kubeconfigs = append(kubeconfigs, config)
	}
    ...
}
```

有两种方式可以获取kubeconfig配置路径：

1、文件路径（即rules.ExplicitPath）

2、环境变量(KUBECONIFG，即rules.Precedence，可指定多个路径)

最终将信息汇总到`kubeConfigFiles`中，这两种方式都通过`LoadFromFile`函数读取数据并把读取到的数据反序列化到Config对象中

```go
func Load(data []byte) (*clientcmdapi.Config, error) {
	config := clientcmdapi.NewConfig()
	// if there's no data in a file, return the default object instead of failing (DecodeInto reject empty input)
	if len(data) == 0 {
		return config, nil
	}
	decoded, _, err := clientcmdlatest.Codec.Decode(data, &schema.GroupVersionKind{Version: clientcmdlatest.Version, Kind: "Config"}, config)
	if err != nil {
		return nil, err
	}
	return decoded.(*clientcmdapi.Config), nil
}
```

### 2、RESEClient客户端

```go
package main

import (
	"fmt"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/rest"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "/root/.kube/config")
	if err != nil {
		panic(err)
	}
    config.APIPath = "api"
    config.GroupVersion = &corev1.SchemeGroupVersion
    config.NegotiatedSerializer = scheme.Codecs

	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err)
	}
    
    result := &core.PodList{}
    err = restClient.Get().Namespace("default").Resource("pods").
    VersionedParams(&metav1.ListOptions{Limit:500},scheme.ParameterCodec).
    Do().Into(result)
    if err != nil {
        panic(err)
    }
    
    for _,d := range result.Items {
        fmt.Printf("NAMESPACE: %v \t NAME: %v \t STATU: %+v\n",d.Namespace,d.Name,d.Status.Phase)
    }
}
```

运行以上代码，列出default命名空间下的所有Pod资源对象的相关信息。首先加载kubeconfig配置信息，并设置config.APIPath请求的HTTP路径。然后设置config.GroupVersion请求的资源组/资源版本。最后设置config.NegotiatedSerializer数据的解码器。

RESTClient发送的过程对Go语言标准库net/http进行了封装，由Do->request函数实现。

[staging/src/k8s.io/client-go/rest/request.go]

```go
func (r *Request) Do(ctx context.Context) Result {
	var result Result
	err := r.request(ctx, func(req *http.Request, resp *http.Response) {
		result = r.transformResponse(resp, req)
	})
	if err != nil {
		return Result{err: err}
	}
	return result
}
```

```go
func (r *Request) request(ctx context.Context, fn func(*http.Request, *http.Response)) error {
	...
	for {
		url := r.URL().String()
		req, err := http.NewRequest(r.verb, url, r.body)
		if err != nil {
			return err
		}
		req = req.WithContext(ctx)
		req.Header = r.headers
         ...
		resp, err := client.Do(req)
		...
		if err != nil {
			if r.verb != "GET" {
				return err
			}
			if net.IsConnectionReset(err) || net.IsProbableEOF(err) {
				resp = &http.Response{
					StatusCode: http.StatusInternalServerError,
					Header:     http.Header{"Retry-After": []string{"1"}},
					Body:       ioutil.NopCloser(bytes.NewReader([]byte{})),
				}
			} else {
				return err
			}
		}

		...
				resp.Body.Close()
		...

			fn(req, resp)
		...
	}
}
```

请求发送之前需要根据请求参数生成请求的RESTful URL，由r.URL.String函数完成。

### 3、ClientSet客户端

RESTClient是最基础的客户端，使用时需要指定Resource和Version等信息。而ClientSet使用更加便捷。

ClientSet在RESTClient的基础上封装了对Resource和Version的管理方法。每个Resource可以理解为一个客户端，而ClientSet则是多个客户端的集合，每个Resource和Version都以函数的形式暴露。

> ClientSet仅能访问kubernetes自身内置的资源（即客户端集合的资源），不能直接访问CRD资源，可以通过client-gen代码生成器重新生成ClientSet，在ClientSet集合中自动生成CRD相关的接口。

```go
package main

import (
	"fmt"
    "k8s.io/client-go/kubernetes"

	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "/root/.kube/config")
	if err != nil {
		panic(err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}
    
    podClient := clientset.CoreV1().Pods(apiv1.NamespaceDefault)
    list,err := podClient.List(metav1.ListOptions{Limit: 500})
    if err != nil {
        panic(err)
    }
    
    for _,d := range list.Items {
        fmt.Printf("NAMESPACE: %v \t NAME: %v \t STATU: %+v\n",d.Namespace,d.Name,d.Status.Phase)
    }
}
```

可以看到对Pod资源执行的操作是在RESTClient进行封装，可以设置选项(如Limit，TimeoutSeconds等)
[staging/src/k8s.io/client-go/kubernetes/typed/core/v1/pod.go]
```go
type pods struct {
	client rest.Interface
	ns     string
}

// newPods returns a Pods
func newPods(c *CoreV1Client, namespace string) *pods {
	return &pods{
		client: c.RESTClient(),
		ns:     namespace,
	}
}

// Get takes name of the pod, and returns the corresponding pod object, and an error if there is any.
func (c *pods) Get(ctx context.Context, name string, options metav1.GetOptions) (result *v1.Pod, err error) {
	result = &v1.Pod{}
	err = c.client.Get().
		Namespace(c.ns).
		Resource("pods").
		Name(name).
		VersionedParams(&options, scheme.ParameterCodec).
		Do(ctx).
		Into(result)
	return
}
```

### 4、DynamicClient客户端

是一种动态客户端，可以对任意kubernetes资源进行RESTClient操作，包括CRD自定义资源。

DynamicClient内部实现了Unstructured，用于处理非结构化数据结构，也是能处理CRD资源的关键。

> DynamicClient不是安全类型，因此在访问CRD自定义资源时需要特别注意。例如，在操作指针不当的情况下可能会导致程序崩溃。

```go
package main

import (
	"fmt"
    apiv1 "k8s.io/api/core/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/dynamic"
    _ "k8s.io/client-go/plugin/pkg/client/auth"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "/root/.kube/config")
	if err != nil {
		panic(err)
	}

	dynamicClient, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err)
	}
    
    gvr := schema.GroupVersionResource{Version: "v1", Resource: "pods"}
    unstructObj,err := dynamicClient.Resource(gvr).Namespace(apiv1.NamespaceDefault).List(metav1.ListOptions{Limit: 500})
    if err != nil {
        panic(err)
    }

    podList := &corev1.PodList{}
    err = runtime.DefaultUnstructuredConverter.FromUnstructured(unstructObj.UnstructruedContent(),podList)
    if err != nil {
        panic(err)
    }
    for _,d := range podList.Items {
        fmt.Printf("NAMESPACE: %v \t NAME: %v \t STATU: %+v\n",d.Namespace,d.Name,d.Status.Phase)
    }
}
```

### 5、DiscoveryClient客户端

发现客户端，主要用于发现kubernetes API Server所支持的资源组、资源版本、资源信息。
用户可以通过DiscoveryClient来查看所支持的资源组、资源版本、资源信息。
kubectl的api-versions和api-resources命令输出也是通过DiscoveryClient实现的。同样，DiscoveryClient在RESTClient的基础上进行了封装。

DiscoveryClient还可以将信息存储到本地，用于本地缓存(Cache)，以减轻对kubernetes API Server访问的压力。
在运行kubernetes组件的机器上，缓存信息默认存储于`~/.kube/cache`和`~/.kube/http-cache`下。

```go
package main

import (
	"fmt"

	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/discovery"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "/root/.kube/config")
	if err != nil {
		panic(err)
	}

	discoveryClient, err := discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		panic(err)
	}
    
    _,APIResourceList,err := discoveryClient.ServerGroupsAndResources()
    if err != nil {
        panic(err)
    }
    
    for _,list := range APIResourceList {
        gv,err := schema.ParseGroupVersion(list.GroupVersion)
        if err != nil {
           panic(err)
        }
        for _,resource := range list.APIResources {
             fmt.Printf("name: %v, group: %v, version: %v\n",resource.Name,gv.Group,gv.Version)           
        }
    }
}
```

kubernetes API Server暴露出`/api`和`/apis`接口。DiscoveryClient通过RESTClient分别请求`/api`和`/apis`接口，从而获取所支持的资源组、资源版本、资源信息。

[staging/src/k8s.io/client-go/discovery/discovery_client.go]
```go
func (d *DiscoveryClient) ServerGroups() (apiGroupList *metav1.APIGroupList, err error) {
	// Get the groupVersions exposed at /api
	v := &metav1.APIVersions{}
	err = d.restClient.Get().AbsPath(d.LegacyPrefix).Do(context.TODO()).Into(v)
	apiGroup := metav1.APIGroup{}
	if err == nil && len(v.Versions) != 0 {
		apiGroup = apiVersionsToAPIGroup(v)
	}
	if err != nil && !errors.IsNotFound(err) && !errors.IsForbidden(err) {
		return nil, err
	}

	// Get the groupVersions exposed at /apis
	apiGroupList = &metav1.APIGroupList{}
	err = d.restClient.Get().AbsPath("/apis").Do(context.TODO()).Into(apiGroupList)
	if err != nil && !errors.IsNotFound(err) && !errors.IsForbidden(err) {
		return nil, err
	}
	// to be compatible with a v1.0 server, if it's a 403 or 404, ignore and return whatever we got from /api
	if err != nil && (errors.IsNotFound(err) || errors.IsForbidden(err)) {
		apiGroupList = &metav1.APIGroupList{}
	}

	// prepend the group retrieved from /api to the list if not empty
	if len(v.Versions) != 0 {
		apiGroupList.Groups = append([]metav1.APIGroup{apiGroup}, apiGroupList.Groups...)
	}
	return apiGroupList, nil
}
```

#### 本地缓存

默认每10分钟与kubernetes API Server同步一次，同步周期较长，因为资源组、资源版本、资源信息一般很少变动。

[discovery\cached\disk\cached_discovery.go]
```go
// ServerResourcesForGroupVersion returns the supported resources for a group and version.
func (d *CachedDiscoveryClient) ServerResourcesForGroupVersion(groupVersion string) (*metav1.APIResourceList, error) {
	filename := filepath.Join(d.cacheDirectory, groupVersion, "serverresources.json")
	cachedBytes, err := d.getCachedFile(filename)
	// don't fail on errors, we either don't have a file or won't be able to run the cached check. Either way we can fallback.
	if err == nil {
		cachedResources := &metav1.APIResourceList{}
		if err := runtime.DecodeInto(scheme.Codecs.UniversalDecoder(), cachedBytes, cachedResources); err == nil {
			klog.V(10).Infof("returning cached discovery info from %v", filename)
			return cachedResources, nil
		}
	}

	liveResources, err := d.delegate.ServerResourcesForGroupVersion(groupVersion)
	if err != nil {
		klog.V(3).Infof("skipped caching discovery info due to %v", err)
		return liveResources, err
	}
	if liveResources == nil || len(liveResources.APIResources) == 0 {
		klog.V(3).Infof("skipped caching discovery info, no resources found")
		return liveResources, err
	}

	if err := d.writeCachedFile(filename, liveResources); err != nil {
		klog.V(1).Infof("failed to write cache to %v due to %v", filename, err)
	}

	return liveResources, nil
}
```

## Informer机制

在kubernetes系统中，组件通过HTTP协议进行通信，在不依赖任何中间件的情况下需要保证消息的实时性、可靠性、顺序性等，依靠的是Informer机制。
kubernetes的其他组件都是通过client-go的Informer机制与kubernetes API Server进行通信的。

### 1、Informer机制架构设计

![img](../../../../docs/images/informer.png)

在Informer架构设计中，有多个核心组件。

1、Reflector

Reflector用于监控（Watch）指定的kubernetes资源，当监控的资源发生变化时，触发相应的变更事件，并将其资源对象存放到本地DeltaFIFO中。

2、DeltaFIFO

FIFO是一个先进先出队列，具有队列操作的基本方法，而Delta是一个资源对象存储，可以保存资源对象的操作类型。

3、Indexer

存储资源对象并自带索引功能的本地存储，Reflector从DeltaFIFO中将消费出来的资源对象存储至Indexer。
Indexer与ETCD集群中数据完全保持一致。client-go可以很方便地从本地存储中读取相应的资源对象数据，而无须每次从远程ETCD集群中读取。

**Example**

```go
package main

import (
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"log"
	"time"
)

func main() {
	config,err := clientcmd.BuildConfigFromFlags("","/root/.kube/config")
	if err != nil {
		panic(err)
	}

	clientset,err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	stopCh := make(chan struct{})
	defer close(stopCh)

	sharedInformers := informers.NewSharedInformerFactory(clientset,time.Minute)
	informer := sharedInformers.Core().V1().Pods().Informer()

	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			mObj := obj.(v1.Object)
			log.Printf("New Pod Added to Store: %s",mObj.GetName())
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			oObj := oldObj.(v1.Object)
			nObj := newObj.(v1.Object)
			log.Printf("%s Pod Upddated to %s",oObj.GetName(),nObj.GetName())
		},
		DeleteFunc: func(obj interface{}) {
			mObj := obj.(v1.Object)
			log.Printf("Pod Deleted from Store: %s",mObj.GetName())
		},
	})
	informer.Run(stopCh)
}
```

每一个kubernetes资源都实现了Informer机制。每个Informer上都会实现Informer和Lister方法。

[staging/src/k8s.io/client-go/informers/core/v1/pod.go]
```go
type PodInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.PodLister
}
```

Informer又称为Shared Informer，是可以共享使用的。如果同一个Informer被实例化多次，每个Informer使用一个Reflector，
那么会运行过多相同的ListAndWatch，太多重复的序列化和反序列化操作会导致Kubernetes API Server负载过重。

[staging/src/k8s.io/client-go/informers/factory.go]
```go
type sharedInformerFactory struct {
	client           kubernetes.Interface
	namespace        string
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	lock             sync.Mutex
	defaultResync    time.Duration
	customResync     map[reflect.Type]time.Duration

	informers map[reflect.Type]cache.SharedIndexInformer // 使用map结构存放所有informer字段
	// startedInformers is used for tracking which informers have been started.
	// This allows Start() to be called multiple times safely.
	startedInformers map[reflect.Type]bool
}
```

### 2、Reflector

Reflector用于监控指定资源的kubernetes资源，当监控的资源发生变化时，触发相应的事件，并将其资源对象存放到本地缓存DeltaFIFO中。

1、获取资源列表数据

ListAndWatch List在程序的第一次运行时获取该资源下所有的对象数据并将其存放到DeltaFIFO中。

[staging/src/k8s.io/client-go/tools/cache/reflector.go]
```go
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
    options := metav1.ListOptions{ResourceVersion: r.relistResourceVersion()}
	if err := func() error {
		panicCh := make(chan interface{}, 1)
		go func() {
            // 1、用于获取资源下的所有对象数据。获取资源数据由options的ResourceVersion（资源版本号）参数控制，
            // 如果ResourceVersion=0，则表示获取所有pod的资源数据；如果表示非0，则表示根据资源版本号获取。
            // 可以使本地缓存中的数据与ETCD集群中的数据保持一致。
            pager := pager.New(pager.SimplePageFunc(func(opts metav1.ListOptions) (runtime.Object, error) {
                return r.listerWatcher.List(opts)
            }))
			list, paginatedResult, err = pager.List(context.Background(), options)
		}()
		if options.ResourceVersion == "0" && paginatedResult {
			r.paginatedResult = true
		}

		r.setIsLastSyncResourceVersionUnavailable(false) // list was successful
		listMetaInterface, err := meta.ListAccessor(list)
		if err != nil {
			return fmt.Errorf("unable to understand list result %#v: %v", list, err)
		}
        // 2、获取资源版本号，标识当前资源对象的版本号。每次修改当前资源前，kubernetes API Server都会更改ResourceVersion，
        // 使得client-go执行Watch操作时可以根据ResourceVersion来确定当前资源对象是否发生变化
		resourceVersion = listMetaInterface.GetResourceVersion()
        // 3、将资源数据转换成资源对象列表，将runtime.Object对象转换成[]runtime.Object对象。
        // 因为获取的是资源下的所有对象数据
		items, err := meta.ExtractList(list)
		if err != nil {
			return fmt.Errorf("unable to understand list result %#v (%v)", list, err)
		}
        // 4、用于将资源对象列表中的资源对象和资源版本号存储至DeltaFIFO中，并会替换已存在的对象
		if err := r.syncWith(items, resourceVersion); err != nil {
			return fmt.Errorf("unable to sync list result: %v", err)
		}
        // 5、设置最新的资源版本号
		r.setLastSyncResourceVersion(resourceVersion)
		return nil
	}(); err != nil {
		return err
	}
    ...
}
```

而`r.listerWatcher.List`实际上调用Pod Informer的ListFunc函数

[staging/src/k8s.io/client-go/informers/core/v1/pod.go]
```go
func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(context.TODO(), options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
			},
		},
		&corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}
```

2、监控资源对象

监控对象通过HTTP协议与kubernetes API Server建立长连接，接受kubernetes API Server发来的资源变更事件。
Watch操作实现机制使用HTTP协议的分块传输编码（Chunked Transfer Encoding）。当client-go调用kubernetes API Server时，
kubernetes API Server在Response的HTTP Header中设置Transfer-Encoding的值为chunked，表示采用分块传输编码，客户端收到该消息后，
便于服务端进行连接，并等待下一个数据块（即资源的事件信息）。

[staging/src/k8s.io/client-go/tools/cache/reflector.go]
```go
for {
		select {
		case <-stopCh:
			return nil
		default:
		}

		timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
		options = metav1.ListOptions{
			ResourceVersion: resourceVersion,
			TimeoutSeconds: &timeoutSeconds,
			AllowWatchBookmarks: true,
		}

		start := r.clock.Now()
		w, err := r.listerWatcher.Watch(options)

		if err := r.watchHandler(start, w, &resourceVersion, resyncerrc, stopCh); err != nil {
			return nil
		}
	}
```

而`r.listerWatcher.Watch`实际上调用Pod Informer的WatchFunc函数.

watchHandler用于处理资源的变更事件，当触发事件后，将对应的资源对象更新到本地缓存DeltaFIFO中并更新ResourceVersion资源版本号。

3、DeltaFIFO

[staging/src/k8s.io/client-go/tools/cache/delta_fifo.go]
```go
type DeltaFIFO struct {
	// lock/cond protects access to 'items' and 'queue'.
	lock sync.RWMutex
	cond sync.Cond

	// `items` maps a key to a Deltas.
	// Each such Deltas has at least one Delta.
	items map[string]Deltas

	// `queue` maintains FIFO order of keys for consumption in Pop().
	// There are no duplicates in `queue`.
	// A key is in `queue` if and only if it is in `items`.
	queue []string

	// populated is true if the first batch of items inserted by Replace() has been populated
	// or Delete/Add/Update/AddIfNotPresent was called first.
	populated bool
	// initialPopulationCount is the number of items inserted by the first call of Replace()
	initialPopulationCount int

	// keyFunc is used to make the key used for queued item
	// insertion and retrieval, and should be deterministic.
	keyFunc KeyFunc

	// knownObjects list keys that are "known" --- affecting Delete(),
	// Replace(), and Resync()
	knownObjects KeyListerGetter

	// Used to indicate a queue is closed so a control loop can exit when a queue is empty.
	// Currently, not used to gate any of CRED operations.
	closed bool

	// emitDeltaTypeReplaced is whether to emit the Replaced or Sync
	// DeltaType when Replace() is called (to preserve backwards compat).
	emitDeltaTypeReplaced bool
}
```

DeltaFIFO与其他队列的不同之处在于，它会保留所有关于资源对象obj的操作类型，队列中会存在拥有不同操作类型的同一个资源对象，
消费者在处理该资源对象时能够了解该资源对象所发生的事情。

`queue`字段存储资源对象的key，该key可以通过`KeyOf`函数计算得到。
[objkey1,objkey2,objkey3]

`items`字段通过map数据结构的方式存储，value存储的是对象的Deltas数组。
[objkey1:[{"Added",obj1}],objkey2:[{"Deleted",obj2}],objkey3:[{"Sync",obj3}]]

[staging/src/k8s.io/client-go/tools/cache/delta_fifo.go]

- 生产者方法

```go
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
	// 1、计算出资源对象的key
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	// 2、将actionType和资源对象构造成Delta，添加到items中，并通过dedupDeltas函数进行去重操作
	oldDeltas := f.items[id]
	newDeltas := append(oldDeltas, Delta{actionType, obj})
	newDeltas = dedupDeltas(newDeltas)

	if len(newDeltas) > 0 {
		if _, exists := f.items[id]; !exists {
			f.queue = append(f.queue, id)
		}
		// 3、更新构造后的Delta并cond.Broadcast通知所有消费者解除阻塞
		f.items[id] = newDeltas
		f.cond.Broadcast()
	} else {
		// This never happens, because dedupDeltas never returns an empty list
		// when given a non-empty list (as it is here).
		// If somehow it happens anyway, deal with it but complain.
		if oldDeltas == nil {
			klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; ignoring", id, oldDeltas, obj)
			return nil
		}
		klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; breaking invariant by storing empty Deltas", id, oldDeltas, obj)
		f.items[id] = newDeltas
		return fmt.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; broke DeltaFIFO invariant by storing empty Deltas", id, oldDeltas, obj)
	}
	return nil
}
```

- 消费者方法

```go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		// 1、队列中没有数据时，通过cond.Wait阻塞等待数据，只有收到cond.Broadcast时才说明有数据被添加，解除当前阻塞状态。
		for len(f.queue) == 0 {
			// When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
			// When Close() is called, the f.closed is set and the condition is broadcasted.
			// Which causes this loop to continue and return from the Pop().
			if f.closed {
				return nil, ErrFIFOClosed
			}

			f.cond.Wait()
		}
		// 2、如果队列不为空，取出queue头部数据，将对象传入process回调函数，由上层消费者处理
		id := f.queue[0]
		f.queue = f.queue[1:]
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}
		item, ok := f.items[id]
		if !ok {
			// This should never happen
			klog.Errorf("Inconceivable! %q was in f.queue but not f.items; ignoring.", id)
			continue
		}
		delete(f.items, id)
		err := process(item)
		// 3、如果回调函数处理错误，则将该对象重新存储队列
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		// Don't need to copyDeltas here, because we're transferring
		// ownership to the caller.
		return item, err
	}
}
```

回调方法

[staging/src/k8s.io/client-go/tools/cache/shared_informer.go]
```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	// from oldest to newest
	for _, d := range obj.(Deltas) {
		switch d.Type {
		case Sync, Replaced, Added, Updated:
			// 当操作类型是Sync, Replaced, Added, Updated时，将该资源对象存储至Indexer（并发安全存储）
			// 并通过distribute分发到SharedInformer
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}

				isSync := false
				switch {
				case d.Type == Sync:
					// Sync events are only propagated to listeners that requested resync
					isSync = true
				case d.Type == Replaced:
					if accessor, err := meta.Accessor(d.Object); err == nil {
						if oldAccessor, err := meta.Accessor(old); err == nil {
							// Replaced events that didn't change resourceVersion are treated as resync events
							// and only propagated to listeners that requested resync
							isSync = accessor.GetResourceVersion() == oldAccessor.GetResourceVersion()
						}
					}
				}
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, false)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```

- Resync机制

将Indexer本地存储中的资源对象同步到DeltaFIFO中，并将这些资源对象设置成Sync类型。
Resync函数在Reflector中定期执行，执行周期由NewReflector函数传入的resyncPeriod参数设定。
```go
func (f *DeltaFIFO) syncKeyLocked(key string) error {
	obj, exists, err := f.knownObjects.GetByKey(key)
	if err != nil {
		klog.Errorf("Unexpected error %v during lookup of key %v, unable to queue object for sync", err, key)
		return nil
	} else if !exists {
		klog.Infof("Key %v does not exist in known objects store, unable to queue object for sync", key)
		return nil
	}

	// If we are doing Resync() and there is already an event queued for that object,
	// we ignore the Resync for it. This is to avoid the race, in which the resync
	// comes with the previous value of object (since queueing an event for the object
	// doesn't trigger changing the underlying store <knownObjects>.
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	if len(f.items[id]) > 0 {
		return nil
	}

	if err := f.queueActionLocked(Sync, obj); err != nil {
		return fmt.Errorf("couldn't queue object: %v", err)
	}
	return nil
}
```

4、Indexer

是用来存储资源对象并自带索引功能的本地存储，Reflector从DeltaFIFO中将消费出来的资源对象存储至Indexer。
Indexer中的数据与ETCD集群中的数据保持完全一致。

- ThreadSafeMap并发安全存储

是一个内存中的存储，其中的数据并不会写入本地磁盘中，每次的增删改查操作都会加锁，以保证数据的一致性。

[staging/src/k8s.io/client-go/tools/cache/thread_safe_store.go]
```go
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}

	// indexers maps a name to an IndexFunc
	indexers Indexers
	// indices maps a name to an Index
	indices Indices
}
```

- Indexer索引器

在每次增删改查ThreadSafeMap数据时，都会通过`updateIndices`或`deleteFromIndices`函数变更Indexer。

**Example**
```go
package main

import (
	"fmt"
	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/tools/cache"
	"strings"
)
// 1、定义一个索引函数，定义查询出所有Pod资源下Annotation字段的key为users的Pod
func UsersIndexFunc(obj interface{}) ([]string, error) {
	pod := obj.(*v1.Pod)
	usersString := pod.Annotations["users"]
	return strings.Split(usersString, ","), nil
}

func main() {
    // 2、初始化indexer对象，第一个参数是用于计算资源对象的key，第二个参数用于定义索引器
	index := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{"byUsers": UsersIndexFunc})

    // 3、添加3个pod资源对象
	pod1 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{
		Name:        "one",
		Annotations: map[string]string{"users": "ernie,bert"}}}
	pod2 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{
		Name:        "two",
		Annotations: map[string]string{"users": "ernie,oscar"}}}
	pod3 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{
		Name:        "three",
		Annotations: map[string]string{"users": "ernie,elmo"}}}
	index.Add(pod1)
	index.Add(pod2)
	index.Add(pod3)

    // 4、查询匹配的pod列表
	erniePods, err := index.ByIndex("byUser", "ernie")
	if err != nil {
		panic(err)
	}

	for _, pod := range erniePods {
		fmt.Println(pod.(*v1.Pod).Name)
	}
}
```

[staging/src/k8s.io/client-go/tools/cache/index.go]
```go
// 索引器函数，定义为接受一个资源对象，返回索引结果列表
// IndexFunc knows how to compute the set of indexed values for an object.
type IndexFunc func(obj interface{}) ([]string, error)

// 存储缓存数据，结构是key/val
// Index maps the indexed value to a set of keys in the store that match on that value
type Index map[string]sets.String

// 存储索引器，key为索引器名称，val为索引器实现函数
// Indexers maps a name to a IndexFunc
type Indexers map[string]IndexFunc

// 存储缓存器，key为缓存区名称，val为换粗数据
// Indices maps a name to an Index
type Indices map[string]Index
```

[staging/src/k8s.io/client-go/tools/cache/thread_safe_store.go]
```go
// ByIndex returns a list of the items whose indexed values in the given index include the given indexed value
func (c *threadSafeMap) ByIndex(indexName, indexedValue string) ([]interface{}, error) {
	c.lock.RLock()
	defer c.lock.RUnlock()

	// 1、查找指定的索引器函数并执行
	indexFunc := c.indexers[indexName]
	if indexFunc == nil {
		return nil, fmt.Errorf("Index with name %s does not exist", indexName)
	}

	// 2、查找指定的缓存器函数
	index := c.indices[indexName]

	// 3、检索indexedValue从缓存数据中查到并返回数据
	set := index[indexedValue]
	list := make([]interface{}, 0, set.Len())
	for key := range set {
		list = append(list, c.items[key])
	}

	return list, nil
}
```

## WorkQueue

工作队列，主要功能在于标记和去重。

- 有序：按照添加顺序处理元素（item）
- 去重：相同元素在同一时间不会被重复处理，例如一个元素在处理之前被添加了很多次，它只会被处理一次
- 并发性：多生产者和多消费者
- 标记机制：支持标记功能，标记一个元素是否被处理，也允许元素在处理时重新排队
- 通知机制：ShutDown方法通过信号量通知队列不再接受新的元素，并通知metric goroutine退出
- 延迟：支持延迟队列，延迟一段时间后再将元素存入队列
- 限速：支持限速队列，元素存入队列时进行速率限制。限制一个元素被重新排队（Reenqueued）的次数
- Metric：支持metric监控指标，可用于Prometheus监控

WorkQueue支持3种队列，并提供3种接口
- Interface：FIFO队列接口，先进先出队列，并支持去重机制
- DelayingInterface：延迟队列接口，基于Interface接口封装，延迟一段时间后再讲元素存入队列
- RateLimitingInterface：限速队列接口，基于DelayingInterface接口封装，支持元素存储队列时进行限速限制

### 1、FIFO队列

[staging/src/k8s.io/client-go/util/workqueue/queue.go]
```go
type Interface interface {
	Add(item interface{})   // 给队列添加元素，可以是任意类型元素
	Len() int               // 返回当前队列的长度
	Get() (item interface{}, shutdown bool) // 获取队列头部的一个元素
	Done(item interface{})  // 标记队列中该元素已被处理
	ShutDown()              // 关闭队列
	ShuttingDown() bool     // 查询队列是否关闭
}

type Type struct {
	queue []t
	dirty set
	processing set
	cond *sync.Cond
	shuttingDown bool
	metrics queueMetrics
	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.Clock
}
```

FIFO队列数据结构中最主要的字段有queue、dirty、processing。
其中queue是实际存储元素的地方，是slice结构，保证元素有序。
dirty除了去重，还能保证在处理一个元素之前哪怕被添加多次（并发情况），但也只会被处理一次。
processing用于标记机制，标记一个元素是否正在被处理。

### 2、延迟队列

基于FIFO队列封装接口，增加了AddAfter方法，其原理是延迟一段时间后再讲元素插入FIFO队列。

[staging/src/k8s.io/client-go/util/workqueue/delaying_queue.go]
```go
type DelayingInterface interface {
	Interface
	// AddAfter adds an item to the workqueue after the indicated duration has passed
	AddAfter(item interface{}, duration time.Duration)
}

type delayingType struct {
	Interface

	// clock tracks time for delayed firing
	clock clock.Clock

	// stopCh lets us signal a shutdown to the waiting loop
	stopCh chan struct{}
	// stopOnce guarantees we only signal shutdown a single time
	stopOnce sync.Once

	// heartbeat ensures we wait no more than maxWait before firing
	heartbeat clock.Ticker

	// waitingForAddCh is a buffered channel that feeds waitingForAdd
	waitingForAddCh chan *waitFor

	// metrics counts the number of retries
	metrics retryMetrics
}
```

`delayingType`最主要的字段是`waitingForAddCh`，其默认初始大小是1000，通过AddAfter方法插入元素时，是非阻塞状态的，只有当插入的元素大于或等于1000时，
延迟队列才会处于阻塞状态。`waitingForAddCh`字段中的数据通过goroutine运行的`waitingLoop`函数持久运行。

### 3、限速队列

基于FIFO和延迟队列接口封装。

[staging/src/k8s.io/client-go/util/workqueue/default_rate_limiters.go]
```go
type RateLimiter interface {
	// When gets an item and gets to decide how long that item should wait
	When(item interface{}) time.Duration
	// Forget indicates that an item is finished being retried.  Doesn't matter whether its for perm failing
	// or for success, we'll stop tracking it
	Forget(item interface{})
	// NumRequeues returns back how many failures the item has had
	NumRequeues(item interface{}) int
}
```

**4中限速算法**

- 令牌桶算法（BucketRateLimiter）

通过第三方包[golang.org/x/time/rate]实现。令牌桶算法内部实现了一个存放token（令牌）的“桶”，初始时“桶”是空的，token会以固定速率往“桶”里填充，直到将其填满为止，
多余的token会被丢弃。每个元素都会从令牌桶得到一个token，只有得到token的元素才会允许通过（accept），而没有得到token的元素处于等待状态。令牌桶通过控制发放token来达到限速目的。

```go
rate.NewLimiter(rate.Limit(10), 100)
```

实例化rate.NewLimiter后，传入r、b两个参数，r表示每秒往“桶”里填充的token数量，r表示 令牌桶的大小。

假定r=10，b=100，那么在一个限速周期（1s）里存放1000个元素，通过`r.Limiter.Reserve().Delay()`函数返回元素应该等待的时间。
前100个会被立即处理，之后的元素的延迟时间为item100/100ms,item101/200ms,item102/300ms

- 排队指数算法（ItemExponentialFailureRateLimiter）

排队指数算法将相同元素的排队数作为指数，排队数增大，速率限制呈指数级增长，但其最大值不会超过`maxDelay`。

元素的排队数统计是有限速周期的，一个限速周期是指从执行AddRateLimited方法到执行完Forget方法之间的时间。如果该元素的Forget方法处理完，则清空排队数。
```go
    r.failures[item] = r.failures[item] + 1

	// The backoff is capped such that 'calculated' value never overflows.
	backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
	if backoff > math.MaxInt64 {
		return r.maxDelay
	}
```

通过延迟多个相同元素的插入时间，达到限速的目的。假设一个限速周期内插入10个相同元素，那么第1个元素插入并设置延迟时间为1ms，第2个为2ms，第3个为4ms...，最长不超过1000s

- 计数器算法（ItemFastSlowRateLimiter）

限制一段时间内匀速通过的元素数量。例如在1分钟内允许通过100个元素，每插入一个元素，计数器自增1，当计数器到100的阈值且还在限速周期内时，不允许元素再通过。

这里在此基础上扩展了fast和slow速率。例如fastDelay为5ms，slowDelay为10ms，在一个限速周期内插入4个元素，前3个使用fastDelay定义的fast速率，
当触发maxFastAttempts字段，第4个元素使用slowDelay定义的slow速率。

- 混合模式（MaxOfRateLimiter），将多重限速算法混合使用

## EventBroadcaster事件管理器

kubernetes的事件Event是一个资源对象，用于展示集群内发生的情况，kubernetes系统的各个组件会将运行时发生的各种事件上报给kubernetes API Server。

事件存储在ETCD集群中，为避免磁盘空间被填满，强制执行保留策略：在最后一次的事件发生后，删除1小时之前发生的事件。

[staging/src/k8s.io/api/core/v1/types.go]
```go
type Event struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	metav1.ObjectMeta `json:"metadata" protobuf:"bytes,1,opt,name=metadata"`

	// The object that this event is about.
	InvolvedObject ObjectReference `json:"involvedObject" protobuf:"bytes,2,opt,name=involvedObject"`

	// This should be a short, machine understandable string that gives the reason
	// for the transition into the object's current status.
	// TODO: provide exact specification for format.
	// +optional
	Reason string `json:"reason,omitempty" protobuf:"bytes,3,opt,name=reason"`

	// A human-readable description of the status of this operation.
	// TODO: decide on maximum length.
	// +optional
	Message string `json:"message,omitempty" protobuf:"bytes,4,opt,name=message"`

	// The component reporting this event. Should be a short machine understandable string.
	// +optional
	Source EventSource `json:"source,omitempty" protobuf:"bytes,5,opt,name=source"`

	// The time at which the event was first recorded. (Time of server receipt is in TypeMeta.)
	// +optional
	FirstTimestamp metav1.Time `json:"firstTimestamp,omitempty" protobuf:"bytes,6,opt,name=firstTimestamp"`

	// The time at which the most recent occurrence of this event was recorded.
	// +optional
	LastTimestamp metav1.Time `json:"lastTimestamp,omitempty" protobuf:"bytes,7,opt,name=lastTimestamp"`

	// The number of times this event has occurred.
	// +optional
	Count int32 `json:"count,omitempty" protobuf:"varint,8,opt,name=count"`

	// Type of this event (Normal, Warning), new types could be added in the future
	// +optional
	Type string `json:"type,omitempty" protobuf:"bytes,9,opt,name=type"`

	// Time when this Event was first observed.
	// +optional
	EventTime metav1.MicroTime `json:"eventTime,omitempty" protobuf:"bytes,10,opt,name=eventTime"`

	// Data about the Event series this event represents or nil if it's a singleton Event.
	// +optional
	Series *EventSeries `json:"series,omitempty" protobuf:"bytes,11,opt,name=series"`

	// What action was taken/failed regarding to the Regarding object.
	// +optional
	Action string `json:"action,omitempty" protobuf:"bytes,12,opt,name=action"`

	// Optional secondary object for more complex actions.
	// +optional
	Related *ObjectReference `json:"related,omitempty" protobuf:"bytes,13,opt,name=related"`

	// Name of the controller that emitted this Event, e.g. `kubernetes.io/kubelet`.
	// +optional
	ReportingController string `json:"reportingComponent" protobuf:"bytes,14,opt,name=reportingComponent"`

	// ID of the controller instance, e.g. `kubelet-xyzf`.
	// +optional
	ReportingInstance string `json:"reportingInstance" protobuf:"bytes,15,opt,name=reportingInstance"`
}
```

事件有两种类型，Normal是正常事件，Warning是警告事件。
```go
// Valid values for event types (new types could be added in future)
const (
	// Information only and will not cause any problems
	EventTypeNormal string = "Normal"
	// These events are to warn that something might go wrong
	EventTypeWarning string = "Warning"
)
```

### EventBroadcaster事件管理机制

1、EventRecorder：事件生产者，也称为事件记录器，kubernetes系统组件通过EventRecorder记录关键性事件

[staging/src/k8s.io/client-go/tools/record/event.go]

```go
type EventRecorder interface {
	// 对刚发生的事件进行记录
	Event(object runtime.Object, eventtype, reason, message string)
	// 通过格式化输出事件的格式
	Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{})
	// 功能和eventf一样, 但附加注释annotations
	AnnotatedEventf(object runtime.Object, annotations map[string]string, eventtype, reason, messageFmt string, args ...interface{})
}
```

[staging/src/k8s.io/apimachinery/pkg/watch/mux.go]
```go
func (m *Broadcaster) Action(action EventType, obj runtime.Object) {
	m.incoming <- Event{action, obj}
}
```
Action函数通过goroutine实现异步操作，该函数将事件写入m.incoming中，完成事件生产过程

2、EventBroadcaster：事件消费者，也称为事件广播器。EventBroadcaster消费EventRecorder记录的事件并将其分发给目前所有已连接的broadcasterWatch。
分发的过程有两种机制：非阻塞（Non-Blocking）和阻塞（Blocking）分发机制

```go
// Creates a new event broadcaster.
func NewBroadcaster() EventBroadcaster {
	return &eventBroadcasterImpl{
		Broadcaster:   watch.NewBroadcaster(maxQueuedEvents, watch.DropIfChannelFull),
		sleepDuration: defaultSleepDuration,
	}
}
func NewBroadcaster(queueLength int, fullChannelBehavior FullChannelBehavior) *Broadcaster {
	m := &Broadcaster{
		watchers:            map[int64]*broadcasterWatcher{},
		incoming:            make(chan Event, incomingQueueLength),
		stopped:             make(chan struct{}),
		watchQueueLength:    queueLength,
		fullChannelBehavior: fullChannelBehavior,
	}
	m.distributing.Add(1)
	go m.loop()
	return m
}
// distribute sends event to all watchers. Blocking.
func (m *Broadcaster) distribute(event Event) {
	if m.fullChannelBehavior == DropIfChannelFull {
		for _, w := range m.watchers {
			select {
			case w.result <- event:
			case <-w.stopped:
			default: // Don't block if the event can't be queued.
			}
		}
	} else {
		for _, w := range m.watchers {
			select {
			case w.result <- event:
			case <-w.stopped:
			}
		}
	}
}
```
初始化的过程中会在内部启动goroutine，即`m.loop()`来监控`m.incoming`，并将监控的事件通过`m.distribute`分发给所有已连接的broadcasterWatch。

- 非阻塞分发机制下使用`DropIfChannelFull`，如果`w.result`缓冲区已满，会进入default，事件会丢失
- 阻塞分发机制下使用`WaitIfChannelFull`，如果`w.result`缓冲区已满，会阻塞并等待

3、broadcasterWatch：观察者（Watcher）管理，用于定义事件的处理方式：
- StartLogging：将事件写入日志中
- StartRecordingToSink：上报事件到kubernetes API Server并存储到ETCD集群

依赖于`StartEventWatcher`函数，该函数内部运行一个goroutine，用于不断监控EventBroadcaster来发现事件并调用相关函数对事件进行处理。

StartRecordingToSink函数，kube-scheduler组件将`typedv1core.EventSinkImpl`作为上报的自定义函数。
上报事件有3种方法，分别是Create（Post方法）、Update（Put方法）、Patch（Patch方法）。

例如Create方法：

[staging/src/k8s.io/client-go/kubernetes/typed/core/v1/event_expansion.go]
```go
func (e *events) CreateWithEventNamespace(event *v1.Event) (*v1.Event, error) {
	if e.ns != "" && event.Namespace != e.ns {
		return nil, fmt.Errorf("can't create an event with namespace '%v' in namespace '%v'", event.Namespace, e.ns)
	}
	result := &v1.Event{}
	err := e.client.Post().
		NamespaceIfScoped(event.Namespace, len(event.Namespace) > 0).
		Resource("events").
		Body(event).
		Do(context.TODO()).
		Into(result)
	return result, err
}
```

