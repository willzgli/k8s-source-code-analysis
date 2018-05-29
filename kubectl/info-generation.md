# Info 的生成（未完）




在StreamVisitor 结构体的Visit 方法中，循环读出yaml 文件数据，生成了Info对象

先回顾一下StreamVisitor 结构体初始化情况。

```
*StreamVisitor: &StreamVisitor{  //输入流
		                      Reader: nil,
		                      Mapper: b.mapper (b.unstructured)  // *Mapper 
		                      Source: consSTDINstr,
		                      Schema: schema,
		                     }
```

Mapper 属性赋值为b.mapper , b.mapper 即为b.unstructured, b.unstructured 赋值为：

```
 unstructured: &resource.Mapper{
		RESTMapper:   mapper,  //此处 mapper为lazyObject 结构体实例 
		ObjectTyper:  typer,   // lazyObject 结构体实例	
	
		/*unstructuredClientMapperFunc 为f.objectMappingFactory.UnstructuredClientForMapping()            方法*/
		ClientMapper: unstructuredClientMapperFunc,  
		
		/* UnstructuredJSONScheme结构体，有用Decode、Encode、decode、encode 等多个实例方法
		结构体具体声明位于k8s.io/apimachinery/pkg/apis/meta/v1/unstructed.go */
		Decoder:      unstructured.UnstructuredJSONScheme 
		},
```



pkg/kubeclt/resource/mapper.go

```
func (m *Mapper) InfoForData(data []byte, source string) (*Info, error) {

     //Decode 方法为 unstructured 结构体中Decoder 属性具有的方法
	obj, gvk, err := m.Decode(data, nil, nil) 
	if err != nil {
		return nil, fmt.Errorf("unable to decode %q: %v", source, err)
	}

    /*m 即b.mapper（b.unstructed),此处RESTMapping 为在lazyObject中实现的方法。调用过程见下文分析1
    */
	mapping, err := m.RESTMapping(gvk.GroupKind(), gvk.Version)  //分析一
	if err != nil {
		return nil, fmt.Errorf("unable to recognize %q: %v", source, err)
	}
     
    // m.ClientMapping 实际调用UnstructuredClientForMapping() 方法 ,下文分析2
	client, err := m.ClientForMapping(mapping) // 分析二
	if err != nil {
		return nil, fmt.Errorf("unable to connect to a server to handle %q: %v", mapping.Resource, err)
	}

	name, _ := mapping.MetadataAccessor.Name(obj)
	namespace, _ := mapping.MetadataAccessor.Namespace(obj)
	resourceVersion, _ := mapping.MetadataAccessor.ResourceVersion(obj)

	return &Info{
		Client:  client,
		Mapping: mapping,

		Source:          source,
		Namespace:       namespace,
		Name:            name,
		ResourceVersion: resourceVersion,

		Object: obj,
	}, nil
}
```



分析1：mapping, err := m.RESTMapping(gvk.GroupKind(), gvk.Version) 

staging/src/k8s.io/apimachinery/pkg/api/meta/lazy.go

```
func (o *lazyObject) RESTMapping(gk schema.GroupKind, versions ...string) (*RESTMapping, error) {
	if err := o.init(); err != nil {
		return nil, err
	}
	/* o.mapper 在o.init 中赋值,o.mapper 为shortcutExpander，此处RESTMapping 为在shortcutExpander	      实现的方法 */
	return o.mapper.RESTMapping(gk, versions...)  分析三 
}

func (o *lazyObject) init() error {
	o.lock.Lock()
	defer o.lock.Unlock()
	if o.loaded {
		return o.err
	}
	// 回顾Visitor的诞生一节，知道load() 方法即为objectLoader() 方法
	o.mapper, o.typer, o.err = o.loader() // 分析四
	o.loaded = true
	return o.err
}
```



分析四: 进入objectLoader() 方法

```
pkg/kubectl/cmd/uitl/factory_object_mapping.go

func (f *ring1Factory) objectLoader() (meta.RESTMapper, runtime.ObjectTyper, error) {
     
     // discoveryClient 一个CachedDiscoveryClient 实例引用,发现server端支持的API groups，versions 	    and resources */
	discoveryClient, err := f.clientAccessFactory.DiscoveryClient() //分析五
	if err != nil {
		glog.V(3).Infof("Unable to get a discovery client to find server resources, falling back to hardcoded types: %v", err)
		return legacyscheme.Registry.RESTMapper(), legacyscheme.Scheme, nil
	}

	groupResources, err := discovery.GetAPIGroupResources(discoveryClient)
	if err != nil && !discoveryClient.Fresh() {
		discoveryClient.Invalidate()
		groupResources, err = discovery.GetAPIGroupResources(discoveryClient)
	}
	if err != nil {
		glog.V(3).Infof("Unable to retrieve API resources, falling back to hardcoded types: %v", err)
		return legacyscheme.Registry.RESTMapper(), legacyscheme.Scheme, nil
	}

	/intefaces 接收一个匿名函数
	interfaces := meta.InterfacesForUnstructuredConversion(legacyscheme.Registry.InterfacesFor)
	
	/* 返回DeferredDiscoveryRESTMapper 结构体引用，其属性cl 赋值为discoveryClient。 
	DeferredDiscoveryRESTMapper 结构体同样实现了meta.RESTMapper 接口 */
	mapper := discovery.NewDeferredDiscoveryRESTMapper(discoveryClient, meta.VersionInterfacesFunc(interfaces)) 
	
	// TODO: should this also indicate it recognizes typed objects?
	typer := discovery.NewUnstructuredObjectTyper(groupResources, legacyscheme.Scheme)
	
	/* expander 为shortcutExpander 结构体类型，属性RESTMapper 赋值为mapper,需要注意的是
	  mapper 自己实现了meta.RESTMapper 接口种的方法，同时shortcutExpander也实现了meta.RESTMapper接
	  口*/
	expander := NewShortcutExpander(mapper, discoveryClient)
	
	return expander, typer, err 
}
```



分析五：  f.clientAccessFactory.DiscoveryClient()  方法调用

pkg/kubectl/cmd/util/factory_client_access.go
```
func (f *discoveryFactory) DiscoveryClient() (discovery.CachedDiscoveryInterface, error) {
	cfg, err := f.clientConfig.ClientConfig()
	if err != nil {
		return nil, err
	}

	cfg.CacheDir = f.cacheDir
 
     // 基于配置文件创建DiscoveryClient的一个实例，discoveryClient是实例引用
	discoveryClient, err := discovery.NewDiscoveryClientForConfig(cfg) //分析六
	if err != nil {
		return nil, err
	}
	
	/*cacheDir 的值是:  /root/.kube/cache/discovery/localhost_8080
	  或/root/.kube/cache/discovery/kubernetes_6443 */
	cacheDir := computeDiscoverCacheDir(filepath.Join(homedir.HomeDir(), ".kube", "cache", "discovery"), cfg.Host) 
	
	// 返回一个CachedDiscoveryClient 结构体实例引用
	return NewCachedDiscoveryClient(discoveryClient, cacheDir, time.Duration(10*time.Minute)), nil
}  
```



分析六：discovery.NewDiscoveryClientForConfig(cfg) 方法调用

staging/src/k8s.io/client-go/discovery/discovery-client.go

```
func NewDiscoveryClientForConfig(c *restclient.Config) (*DiscoveryClient, error) {
	config := *c
	if err := setDiscoveryDefaults(&config); err != nil {
		return nil, err
	}
	/*client实例类型是RESTClient 结构体引用，在RESTClient 结构体中，包含了一个httpClient 实例，并且
	 实现了POST(),PUT(),Patch(),Get(),Delete（) 等方法
	client, err := restclient.UnversionedRESTClientFor(&config) 
	return &DiscoveryClient{restClient: c, LegacyPrefix: "/api"}, err
}
```



DiscoveryClient() 执行结束，返回objectLoader()，继续执行discovery.NewDeferredDiscoveryRESTMapper()，该方法返回NewDeferredDiscoveryRESTMapper 实例引用。继续执行NewShortcutExpander，返回一个shortcutExpander结构体实例。至此，objectLoader() 方法执行结束

看完了 objectLoader() 方法，总结一下 httpClient 实例的包装过程大致如下：

```
// RESTClient 结构体实例实现了了POST(),PUT(),Patch(),Get(),Delete（) 等方法
client := &RESTClient{  
		base:             &base,
		versionedAPIPath: versionedAPIPath,
		contentConfig:    config,
		serializers:      *serializers,
		createBackoffMgr: readExpBackoffConfig,
		Throttle:         throttle,
		Client:           client, // httpClient 实例
	}

discoveryClient := &DiscoveryClient{restClient: c, //RESTClient 实例引用，即client。
                    LegacyPrefix: "/api"}
                    
                  
discoveryClient := &CachedDiscoveryClient{ 
		    delegate:       delegate,  // DiscoveryClient 实例引用，即discoveryClient，
		    cacheDirectory: cacheDirectory, // 缓存目录
		    ttl:            ttl, // 缓存有效期，传入的值是10*time.Minute
		    ourFiles:       map[string]struct{}{},
		    fresh:          true,
		   }  
		   
// DeferredDiscoveryRESTMapper 	 结构体实现了meta.RESTMapper 接口	   
mapper := &DeferredDiscoveryRESTMapper{
		cl:               cl, //  CachedDiscoveryClient 实例引用 ，即discoveryClient
		versionInterface: versionInterface,//InterfacesForUnstructuredConversion()返回的匿名方法
	}	
    
//shortcutExpander  结构体实现了meta.RESTMapper 接口 
expander :=shortcutExpander{RESTMapper: delegate, // mapper 实例，引用值
                            discoveryClient: client // discoveryClient 实例，引用值
                           }    
```



分析三：再看```o.mapper.RESTMapping(gk, versions...)``` 该条语句的执行。 o.mapper 为 expander 。

```
func (e shortcutExpander) RESTMapping(gk schema.GroupKind, versions ...string) (*meta.RESTMapping, error) {
    // 即调用DeferredDiscoveryRESTMapper的RESTMapping 方法
	return e.RESTMapper.RESTMapping(gk, versions...)
}
```

staging/src/k8s.io/client-go/discovery/restmapper.go

```
func (d *DeferredDiscoveryRESTMapper) RESTMapping(gk schema.GroupKind, versions ...string) (m *meta.RESTMapping, err error) {
	del, err := d.getDelegate() //  del 为PriorityRESTMapper 结构体实例
	if err != nil {
		return nil, err
	}
	m, err = del.RESTMapping(gk, versions...) // 调用PriorityRESTMapper的RESTMapping。 
	if err != nil && !d.cl.Fresh() {
		d.Reset()
		m, err = d.RESTMapping(gk, versions...)
	}
	return
}
```

```
del = meta.PriorityRESTMapper{   // staging/src/k8s.io/client-go/discovery/restmapper.go
		Delegate:         unionMapper, // []RESTMapper 
		ResourcePriority: resourcePriority,
		KindPriority:     kindPriority,
	}
```



最终，```o.mapper.RESTMapping(gk, versions...)``` 返回的

分析2：client, err := m.ClientForMapping(mapping)









