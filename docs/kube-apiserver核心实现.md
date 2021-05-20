kubernetes pod资源对象创建流程：

1、使用kubectl工具向kubernetes API Server发起创建Pod资源对象的请求

2、kubernetes API Server验证请求并将其持久化保存到Etcd集群中

3、kubernetes API Server基于Watch机制通知kube-scheduler调度器

4、kube-scheduler调度器根据预选和优选调度算法为pod资源对象选择最优的节点并通知kubernetes API Server

5、kubernetes API Server将最优节点持久保存到Etcd集群中

6、kubernetes API Server将通知最优节点上的kubelet组件 

7、kubelet组件在所在节点上通过与容器进程交互创建容器

8、kubelet组件将容器状态上报至kubernetes API Server

9、kubernetes API Server将容器状态持久保存到Etcd集群中

## 一、概念

### 1、go-RESTful核心原理

在kubernetes源码中使用了go-restful框架，主要原因在于可定制成都高。

go-restful框架支持多个Container（容器），一个Container就相当于一个HTTP Server，不同Container之间监控不同的地址和端口，对外提供不同的HTTP服务/
每个Container可以包含多个WebService（Web服务），WebService相当于一组不同服务的分类，每个WebService下又包含多个Router（路由），Router根据HTTP请求的URL路由到对应的处理函数。

go-restful核心数据结构有Container、WebService、Route。核心原理是将Container接收到的HTTP请求分发给匹配的WebService，再由WebService分发给Router中的Handler。

[vendor/github.com/emicklei/go-restful/container.go]
```go
// Dispatch the incoming Http Request to a matching WebService.
func (c *Container) dispatch(httpWriter http.ResponseWriter, httpRequest *http.Request) {
	...
	// Find best match Route ; err is non nil if no match was found
	var webService *WebService
	var route *Route
	var err error
	func() {
		c.webServicesLock.RLock()
		defer c.webServicesLock.RUnlock()
		webService, route, err = c.router.SelectRoute(c.webServices, httpRequest)
	}()
	...
	pathProcessor, routerProcessesPath := c.router.(PathProcessor)
	if !routerProcessesPath {
		pathProcessor = defaultPathProcessor{}
	}
	pathParams := pathProcessor.ExtractParameters(route, webService, httpRequest.URL.Path)
	wrappedRequest, wrappedResponse := route.wrapRequestResponse(writer, httpRequest, pathParams)
	// pass through filters (if any)
	if len(c.containerFilters)+len(webService.filters)+len(route.Filters) > 0 {
		// compose filter chain
		allFilters := []FilterFunction{}
		allFilters = append(allFilters, c.containerFilters...)
		allFilters = append(allFilters, webService.filters...)
		allFilters = append(allFilters, route.Filters...)
		chain := FilterChain{Filters: allFilters, Target: func(req *Request, resp *Response) {
			// handle request by route after passing all filters
			route.Function(wrappedRequest, wrappedResponse)
		}}
		chain.ProcessFilter(wrappedRequest, wrappedResponse)
	} else {
		// no filters, handle request by route
		route.Function(wrappedRequest, wrappedResponse)
	}
}
```

dispatch函数进行分发的过程可分为两步：

1、通过c.router.SelectRoute根据请求匹配到最优的WebService->Router，其中包含两种RouteSelector，分别是RouterJSR311和CurlyRouter（基于RouterJSR311实现，支持正则表达式和动态参数，默认使用）

2、根据请求路径调用对应的Handler函数，执行route.Function函数

**Example**

```go
package main

import (
	"github.com/emicklei/go-restful/v3"
	"io"
	"log"
	"net/http"
)

func main() {
	ws := new(restful.WebService)
	//ws.Path("/hello").Consumes(restful.MIME_XML,restful.MIME_JSON).Produces(restful.MIME_JSON,restful.MIME_XML)

	ws.Route(ws.GET("/hello").To(hello))
	restful.Add(ws)
	log.Fatal(http.ListenAndServe(":8080",nil))
}

func hello(request *restful.Request, response *restful.Response) {
	io.WriteString(response,"world\n")
}
```

### 2、一次HTTP请求的完整生命周期

1、用户向kubernetes API Server发出HTTP请求

2、kubernetes API Server接收到用户发出的请求

3、kubernetes API Server启用goroutine处理接收到的请求

4、kubernetes API Server验证请求内容中的认证（auth）信息

5、kubernetes API Server解析请求内容

6、kubernetes API Server调用路由项对应的Handler回调函数

7、kubernetes API Server获取Handler回调函数的数据信息

8、kubernetes API Server设置请求状态码

9、kubernetes API Server响应用户的请求

### 3、OpenAPI/Swagger核心原理

Swagger是OpenAPI规范的落地实现，通过Swagger可以基于OpenAPI规范的REST API，从文档生成、编辑、测试到各种主流语言的代码自动生成都能完成。

### 4、HTTPS核心原理

HTTP协议：超文本传输协议，客户端浏览器或其他应用程序与Web服务器之间的应用层通信协议。

HTTPS在HTTP协议上增加一层TLS/SSL安全基础层，用于在HTTP协议上安全地传输数据。

HTTPS协议与HTTP协议相比具有如下特征：
- 所有信息都是加密传输的，第三方无法窃听
- 具有校验数据完整性机制，确保数据在传输过程中不被篡改
- 配备身份整数，防止身份造假

**HTTPS加密传输过程**

1、客户端首次请求服务端，告诉服务端自己支持的协议版本、支持的加密算法及压缩算法，并生成一个客户端随机数（Client Random）并告知服务端

2、服务端确认双方使用的加密算法，并返回给客户端整数及一个服务端生成的服务端随机数（Server Random）

3、客户端收到证书后，首先验证证书的有效性，然后生成一个新的随机数（即Premaster Secret），使用数字证书中的公钥来加密这个随机数，并发送给服务端

4、服务端接收到已加密的随机数后，使用私钥进行解密，获取这个随机数（即Premaster Secret）

5、服务端和客户端根据约定的加密算法，使用前面3个随机数（Client Random、Server Random、Premaster Secret），生成对话秘钥（Session Key），用来加密接下来的整个对话过程。

### 5、gRPC核心原理

gRPC分为两部分，分别是RPC框架和数据序列化（Data Serialization）框架

1、gRPC是一个超快速、超高效的RPC服务，可以在许多主流的编程语言、平台、框架中轻松创建高性能、可扩展的API和微服务。

2、gRPC默认使用Protocol Buffers作为IDL（Interface Description Language）语言，Protocol Buffers主要用于结构化数据的序列化和反序列化。
Protocol Buffers是一种高性能的开源二进制序列化协议，可以轻松定义服务并自动生成客户端。

gRPC基于HTTP/2协议标准实现，复用了很多HTTP/2的特性，例如双向流、流控制、报头压缩，以及通过单个TCP连接的多路复用请求等。HTTP/2可以使程序更快、更简单、更强大。
另外，Protocol Buffers协议在数据帧上内置了HTTP/2的流控制，这样用户可以方便地控制系统吞吐量，但在诊断架构中的问题时会增加额外的复杂性，因为客户端和服务端都可以设置自己的流量控制值。

**gRPC跨语言通信流程**

1、客户端（gRPC Stub）调用A方法，发起RPC调用

2、对请求信息使用Protobuf进行对象序列化压缩（IDL）

3、服务端（gRPC Server）接收到请求后，解码请求体，进行业务逻辑处理并返回

4、对响应结果使用Protobuf进行对象序列化压缩（IDL）

5、客户端接收到服务端响应，解码请求体。回调被调用的方法A，唤醒正在等待响应（阻塞）的客户端调用并返回响应结果

### 6、go-to-protobuf代码生成器

go-to-protobuf是一种为资源生成Protobuf IDL（即*.proto文件）文件的工具。Protobuf IDL主要用于结构化数据的序列化和反序列化。

#### 1、基础类Tags

基础类Tags决定是否生成Protobuf相关代码
```go
// +protobuf=true   // 为当前的struct生成代码
// +protobuf=false  // 不为当前的struct生成代码
// +protobuf.nullable=true // 为当前struct结构体生成指针的字段（只在具有types.Map和types.Slice的struct生效）
```

#### 2、引用类Tags

引用类Tags（protobuf.as）可以应用另一个struct，并为其生成代码
```go
// +protobuf.as=Timestamp
```
protobuf.as应用了Timestamp结构体，并同时为Timestamp生成代码

[staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/time.go]
```go
// +protobuf.as=Timestamp
type Time struct {
	time.Time `protobuf:"-"`
}
```

[staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/time_proto.go]
```go
type Timestamp struct {
	Seconds int64 `json:"seconds" protobuf:"varint,1,opt,name=seconds"`
	Nanos int32 `json:"nanos" protobuf:"varint,2,opt,name=nanos"`
}
```
Time结构体通过protobuf.as引入Timestamp结构体，通过go-to-protobuf代码生成器生成代码，生成后的代码如下

[staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/generated.proto]
```proto
message Timestamp {
  optional int64 seconds = 1;
  optional int32 nanos = 2;
}
```

#### 3、嵌入类Tags

嵌入类Tags(protobuf.embed)可以嵌入一个类型，并为其生成代码。
```go
// +protobuf.embed=string
```
protobuf.embed指定了string类型，生成代码时，将其指定为string类型。

[staging/src/k8s.io/apimachinery/pkg/api/resource/quantity.go]
```go
// +protobuf.embed=string
type Quantity struct {
	i int64Amount
	d infDecAmount
	s string
	Format
}
```
生成后代码如下：

[staging/src/k8s.io/apimachinery/pkg/api/resource/generated.proto]
```proto
message Quantity {
  optional string string = 1;
}
```

#### 4、选项类Tags

选项类Tags（protobuf.options）可以设置生成的结构体。
```go
// +protobuf.options.marshal=false
// +protobuf.options.(gogoproto.goproto_stringer)=false
```
protobuf.options.(gogoproto.goproto_stringer)的默认值为true，当其为false时，代码生成器生成代码时不为当前struct生成string方法。

当protobuf.options.marshal为false时，代码生成器生成代码时不为当前struct生成Marshal\MarshalTo\Size\Unmarshal方法

### go-to-protobuf的使用示例

go-to-protobuf代码生成器依赖于外部的protoc二进制文件。

## 二、kube-apiserver架构设计

kube-apiserver提供了3种HTTP Server：
- APIExtensionsServer：API扩展服务（扩展器）。该服务提供CRD（CustomResourceDefinitions）自定义资源服务，开发者可通过CRD对kubernetes资源进行扩展。
服务通过CRD对象进行管理，并通过extensionsapiserver.Scheme资源注册表管理CRD相关资源。
- AggregatorServer：API聚合服务（聚合器）。该服务提供AA（APIAggregator）聚合服务，开发者可通过AA对kubernetes聚合服务进行扩展，例如metrics-server是kubernetes系统集群的核心监控数据的聚合器，是AggregatorServer服务的扩展实现。
API聚合服务通过APIAggregator对象进行管理，并通过aggregatorscheme.Scheme资源注册表管理AA相关资源。
- KubeAPIServer：API核心服务。该服务提供kubernetes内置核心资源服务，不允许开发者随意更改相关资源。
API核心服务通过Master对象进行管理，并通过legacyscheme.Scheme资源注册表管理Master相关资源。

它们底层都依赖于GenericApiServer，通过GenericApiServer可以将kubernetes资源与REST API 进行映射。

## 三、kube-apiserver启动流程

### 1、资源注册

kube-apiserver组件启动的第一件事是将kubernetes所支持的资源注册到Scheme资源注册表中。资源注册时通过Go语言的Import和初始化（init）机制触发的。

[cmd/kube-apiserver/app/server.go]
```go
import (
    "k8s.io/kubernetes/pkg/api/legacyscheme"
    "k8s.io/kubernetes/pkg/controlplane"
)
```

1、初始化Scheme注册表资源

[pkg/api/legacyscheme/scheme.go]

```go
var (
	Scheme = runtime.NewScheme()
	Codecs = serializer.NewCodecFactory(Scheme)
	ParameterCodec = runtime.NewParameterCodec(Scheme)
)
```
在`legacyscheme`包中，定义了Scheme注册表、Codec编解码器及ParameterCodec参数编解码器。

2、注册kubernetes所支持的资源

["k8s.io/kubernetes/pkg/controlplane"](cmd/kube-apiserver/app/server.go) -> [corerest "k8s.io/kubernetes/pkg/registry/core/rest"](pkg/controlplane/controller.go) ->
["k8s.io/apiserver/pkg/registry/rest"](pkg/registry/core/rest/storage_core.go) -> [printersinternal "k8s.io/kubernetes/pkg/printers/internalversion"](pkg/registry/core/componentstatus/rest.go)

[pkg/printers/internalversion/import_known_versions.go]

```go
import (
	// These imports are the API groups the client will support.
	// TODO: Remove these manual install once we don't need legacy scheme in get comman
	_ "k8s.io/kubernetes/pkg/apis/apps/install"
	_ "k8s.io/kubernetes/pkg/apis/authentication/install"
	_ "k8s.io/kubernetes/pkg/apis/authorization/install"
	_ "k8s.io/kubernetes/pkg/apis/autoscaling/install"
	_ "k8s.io/kubernetes/pkg/apis/batch/install"
	_ "k8s.io/kubernetes/pkg/apis/certificates/install"
	_ "k8s.io/kubernetes/pkg/apis/coordination/install"
	_ "k8s.io/kubernetes/pkg/apis/core/install"
	_ "k8s.io/kubernetes/pkg/apis/discovery/install"
	_ "k8s.io/kubernetes/pkg/apis/events/install"
	_ "k8s.io/kubernetes/pkg/apis/extensions/install"
	_ "k8s.io/kubernetes/pkg/apis/policy/install"
	_ "k8s.io/kubernetes/pkg/apis/rbac/install"
	_ "k8s.io/kubernetes/pkg/apis/scheduling/install"
	_ "k8s.io/kubernetes/pkg/apis/storage/install"
)
``` 
每个kubernetes内部版本资源都定义install包，用于在kube-apiserver启动时注册资源。
以core资源组为例，会触发install包下的init函数来完成资源的注册过程。

[pkg/apis/core/install/install.go]
```go
func init() {
	Install(legacyscheme.Scheme)
}

// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
	utilruntime.Must(core.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion))
}
```

### 2、Cobra命令行参数解析

```go
func NewAPIServerCommand() *cobra.Command {
	s := options.NewServerRunOptions()
	cmd := &cobra.Command{
		RunE: func(cmd *cobra.Command, args []string) error {
			verflag.PrintAndExitIfRequested()
			fs := cmd.Flags()
			cliflag.PrintFlags(fs)

			err := checkNonZeroInsecurePort(fs)
			if err != nil {
				return err
			}
			// set default options
			completedOptions, err := Complete(s)
			if err != nil {
				return err
			}

			// validate options
			if errs := completedOptions.Validate(); len(errs) != 0 {
				return utilerrors.NewAggregate(errs)
			}

			return Run(completedOptions, genericapiserver.SetupSignalHandler())
		},
		Args: func(cmd *cobra.Command, args []string) error {
			for _, arg := range args {
				if len(arg) > 0 {
					return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
				}
			}
			return nil
		},
	}
    ...
	return cmd
}
```
首先kube-apiserver组件通过`options.NewServerRunOptions`初始化各个模块的默认配置，例如初始化ETCD、Audit、Admission等模块的默认配置。
然后通过Complete函数填充默认的配置参数，并通过Validate函数验证配置参数的合法性和可用性。最后将ServerRunOptions（kube-apiserver组件的运行配置）对象传入Run函数，
Run函数定义了kube-apiserver组件启动的逻辑，它是一个运行不退出的常驻进程。至此，完成了kube-apiserver组件启动之前的环境配置。

### 3、创建APIServer通用配置

Run函数下面

### 4、创建APIExtensionsServer

[New](staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go)

### 5、创建KubeAPIServer

### 6、创建AggregatorServer

### 7、创建GenericAPIServer

无论是APIExtensionsServer、KubeAPIServer，还是AggregatorServer，在底层都依赖于GenericApiServer

[staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go]
```go
genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
```
创建GenericAPIServer，在New->NewAPIServerHandler

[staging/src/k8s.io/apiserver/pkg/server/handler.go]
```go
    gorestfulContainer := restful.NewContainer()
	gorestfulContainer.ServeMux = http.NewServeMux()
	gorestfulContainer.Router(restful.CurlyRouter{})
```
installAPI通过routes注册GenericAPIServer的相关API
- routes.Index：用于获取index索引页面
- routes.Profiling：用于分析性能的可视化页面
- routes.MetricsWithReset：用于获取metrics指标信息，一般用于prometheus指标采集
- routes.Version：用于获取kubernetes系统版本信息

### 8、启动HTTP服务

[staging/src/k8s.io/apiserver/pkg/server/deprecated_insecure_serving.go]
```go
func (s *DeprecatedInsecureServingInfo) Serve(handler http.Handler, shutdownTimeout time.Duration, stopCh <-chan struct{}) error {
	insecureServer := &http.Server{
		Addr:           s.Listener.Addr().String(),
		Handler:        handler,
		MaxHeaderBytes: 1 << 20,
	}

	if len(s.Name) > 0 {
		klog.Infof("Serving %s insecurely on %s", s.Name, s.Listener.Addr())
	} else {
		klog.Infof("Serving insecurely on %s", s.Listener.Addr())
	}
	_, err := RunServer(insecureServer, s.Listener, shutdownTimeout, stopCh)
	// NOTE: we do not handle stoppedCh returned by RunServer for graceful termination here
	return err
}
```

通过标准库`server.Serve`监听listener，并在运行过程中为每个连接创建一个goroutine，goroutine读取请求，然后调用Handler函数来处理并响应请求。
[staging/src/k8s.io/apiserver/pkg/server/secure_serving.go]
```go
func RunServer(
	server *http.Server,
	ln net.Listener,
	shutDownTimeout time.Duration,
	stopCh <-chan struct{},
) (<-chan struct{}, error) {
	if ln == nil {
		return nil, fmt.Errorf("listener must not be nil")
	}
    ...
	go func() {
		err := server.Serve(listener)
	}()

	return stoppedCh, nil
}
```
在这里实现了平滑关闭HTTP服务的功能，利用Go语言标准库的`server.Shutdown`函数可以在不干扰任何活跃连接的情况下关闭服务。
原理是，首选关闭所有的监听listener，然后关闭所有的空闲连接，接着无限期等待苏哟有连接变成空闲状态并关闭。
```go
stoppedCh := make(chan struct{})
	go func() {
		defer close(stoppedCh)
		<-stopCh
		ctx, cancel := context.WithTimeout(context.Background(), shutDownTimeout)
		server.Shutdown(ctx)
		cancel()
	}()
```

### 9、启动HTTPS服务

[staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go]
```go
stoppedCh, err = s.SecureServingInfo.Serve(s.Handler, s.ShutdownTimeout, internalStopCh)
```

[staging/src/k8s.io/apiserver/pkg/server/secure_serving.go]
```go
tlsConfig, err := s.tlsConfig(stopCh)
	if err != nil {
		return nil, err
	}

	secureServer := &http.Server{
		Addr:           s.Listener.Addr().String(),
		Handler:        handler,
		MaxHeaderBytes: 1 << 20,
		TLSConfig:      tlsConfig,
	}
```

## 四、权限控制

- 认证：针对请求的认证，确认是否具有访问kubernetes集群的权限
- 授权：针对资源的授权，确认是否对资源具有相关权限
- 准入控制器：在认证和授权之后，对象被持久化之前，拦截kube-apiserver的请求，拦截后的请求进入准入控制器中处理，对请求的资源对象进行自定义（拦截、修改或拒绝）

## 五、认证

在开启HTTPS服务后，所有的请求都需要经过认证。kube-apiserver支持多种认证机制，并支持同时开启多个认证功能。当客户端发起一个请求，经过认证阶段，只要有一个认证器通过，则认证成功。
如果认证成功，用户名就会传入授权阶段做进一步的授权认证，而对于认证失败的请求则返回401状态码。

### BasicAuth认证

BasicAuth是一种简单的HTTP协议上的认证方式，客户端将用户、密码写入请求头中，HTTP服务端尝试从请求头中验证用户、密码信息，从而实现身份验证。

```text
Authorization: Basic BASE64ENCODED(USER:PASSWORD)
```

#### 1、启用BasicAuth认证

kube-apiserver通过指定--basic-auth-file参数启用BasicAuth认证。AUTH_FILE是一个CSV文件，每个CSV中的表现为password，username，uid

例：123456，derek，1

#### 2、BasicAuth认证接口定义

[staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go]
```go
type Request interface {
	AuthenticateRequest(req *http.Request) (*Response, bool, error)
}
```

### ClientCA认证

ClientCA认证，也被称为TLS双向认证，及服务端和客户端互相验证证书的正确性。使用ClientCA认证认证时，只要是CA签名过的整数都可以通过验证。

#### 1、启动ClientCA认证


kube-apiserver通过指定--client-ca-file参数启动ClientCA认证


#### 2、ClientCA认证接口定义

[staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go]
```go
type Request interface {
	AuthenticateRequest(req *http.Request) (*Response, bool, error)
}
```

#### 3、ClientCA认证实现

[staging/src/k8s.io/apiserver/pkg/authentication/request/x509/x509.go]
```go
func (a *Authenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	if req.TLS == nil || len(req.TLS.PeerCertificates) == 0 {
		return nil, false, nil
	}
    ...
	remaining := req.TLS.PeerCertificates[0].NotAfter.Sub(time.Now())
	clientCertificateExpirationHistogram.Observe(remaining.Seconds())
	chains, err := req.TLS.PeerCertificates[0].Verify(optsCopy)
	if err != nil {
		return nil, false, fmt.Errorf(
			"verifying certificate %s failed: %w",
			certificateIdentifier(req.TLS.PeerCertificates[0]),
			err,
		)
	}
    ...
	return nil, false, utilerrors.NewAggregate(errlist)
}
```

### TokenAuth认证

Token也被称为令牌，服务端为了验证客户端的身份，需要客户端向服务端提供一个可靠的验证信息Token。TokenAuth是基于Token认证。

### BootstrapToken认证

当kubernetes集群中有非常多的节点时，手动为每个节点配置TLS认证比较麻烦，为此kubernetes提供了BootstrapToken认证，也成为引导Token。
客户端的Token信息与服务端的Token相匹配，则认证通过，自动为节点颁发证书，这是一种引导Token的机制

### RequestHeader认证

kubernetes可以设置一个认证代理，客户端发送的认证请求可以通过认证代理将验证信息发送给kube-apiserver组件。
RequestHeader认证使用的就是这种代理方式，它使用请求头将用户名和组信息发送给kube-apiserver。
RequestHeader认证有一个列表：
- 用户名列表：X-Remote-User
- 组列表：X-Remote-Group
- 额外列表：X-Remote-Extra-

### WebhookTokenAuth认证

Webhook称为钩子，是一种基于HTTP协议的回调机制，当客户端发送的认证请求到达kube-apiserver时，kube-apiserver回调钩子方法，将验证信息发送给远程的Webhook服务器进行验证，然后根据Webhook服务器返回的状态码来判断是否认证成功。

### Anonymous认证

匿名认证，未被其他认证服务器拒绝的请求都可视为匿名请求

### OIDC认证

OpenID Connect是一套基于OAuth 2.0协议的轻量级认证规范，其提供了通过API进行身份交互的框架。OIDC认证除了认证请求外，还会表明请求的用户身份（ID Token）

OIDC认证流程如下：

1、kubernetes用户想访问kubernetes API Server，先通过认证服务（Auth Server）认证自己，得到access_token,id_token和refresh_token

2、kubernetes用户把access_token,id_token和refresh_token配置到客户端应用程序上（如kubectl或dashboard工具等）

3、kubernetes客户端使用Token以用户的身份访问kubernetes API Server

### ServiceAccountAuth认证

服务账户令牌。

## 六、授权

## 七、准入控制器

## 八、进程信号处理机制

kubernetes基于UNIX信号（SIGNAL信号）来实现常驻进程及进程的优雅退出。


[staging/src/k8s.io/apiserver/pkg/server/signal.go]
```go
func SetupSignalContext() context.Context {
	close(onlyOneSignalHandler) // panics when called twice

	shutdownHandler = make(chan os.Signal, 2)

	ctx, cancel := context.WithCancel(context.Background())
	signal.Notify(shutdownHandler, shutdownSignals...)
	go func() {
		<-shutdownHandler
		cancel()
		<-shutdownHandler
		os.Exit(1) // second signal. Exit directly.
	}()

	return ctx
}
```

[staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go]
```go
func (s preparedGenericAPIServer) Run(stopCh <-chan struct{}) error {
	delayedStopCh := make(chan struct{})

	go func() {
		defer close(delayedStopCh)

		<-stopCh

		// As soon as shutdown is initiated, /readyz should start returning failure.
		// This gives the load balancer a window defined by ShutdownDelayDuration to detect that /readyz is red
		// and stop sending traffic to this server.
		close(s.readinessStopCh)

		time.Sleep(s.ShutdownDelayDuration)
	}()

	// close socket after delayed stopCh
	stoppedCh, err := s.NonBlockingRun(delayedStopCh)
	if err != nil {
		return err
	}

	<-stopCh

	// run shutdown hooks directly. This includes deregistering from the kubernetes endpoint in case of kube-apiserver.
	err = s.RunPreShutdownHooks()
	if err != nil {
		return err
	}

	// wait for the delayed stopCh before closing the handler chain (it rejects everything after Wait has been called).
	<-delayedStopCh
	// wait for stoppedCh that is closed when the graceful termination (server.Shutdown) is finished.
	<-stoppedCh

	// Wait for all requests to finish, which are bounded by the RequestTimeout variable.
	s.HandlerChainWaitGroup.Wait()

	return nil
}
```
stopCh处于阻塞状态，当按下Ctrl+C时，调用cancel函数，ctx.Done发送信号，退出前完成清理工作。

如果进程被systemd管理，需要向该进程报告当前进程状态。systemd.SdNotify用于守护进程向systemd报告进程状态的变化，其中一项是向systemd报告启动已完成的消息（READY=1）

[staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go]
```go
func (s preparedGenericAPIServer) NonBlockingRun(stopCh <-chan struct{}) (<-chan struct{}, error) {
	...
	if _, err := systemd.SdNotify(true, "READY=1\n"); err != nil {
		klog.Errorf("Unable to send systemd daemon successful start message: %v\n", err)
	}

	return stoppedCh, nil
}
```