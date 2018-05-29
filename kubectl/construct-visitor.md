# **Visitor实例构建**

**Builder实例初始化及处理**

Builder 结构体保存了从命令行获取的各种参数，以及它实现了各种函数用于处理这些参数 并将其转换为一系列的resources，最终用Visitor 方法迭代处理resource。

pkg/kubectl/cmd/util/factory_object_mapping.go

```
func (f *ring2Factory) NewBuilder() *resource.Builder {
	clientMapperFunc := resource.ClientMapperFunc(f.objectMappingFactory.ClientForMapping)
	mapper, typer := f.objectMappingFactory.Object()

	unstructuredClientMapperFunc := resource.ClientMapperFunc(f.objectMappingFactory.UnstructuredClientForMapping)

	categoryExpander := f.objectMappingFactory.CategoryExpander()

	return resource.NewBuilder(...) 
}
```



先看```f.objectMappingFactory.Object()``` 调用。前边说到变量objectMappingFactory 是一个ring1Factory 结构体实例。接下来看一下ring1Factory() 的方法Object()的实现。

pkg/kubectl/cmd/util/factory_object_mapping.go

```
func (f *ring1Factory) Object() (meta.RESTMapper, runtime.ObjectTyper) {
	return meta.NewLazyObjectLoader(f.objectLoader)
}
```

顺藤摸瓜，再看 meta.NewLazyObjectLoader() 方法。

staging/src/k8s.io/apimachinery/pkg/api/meta/lazy.go

```
func NewLazyObjectLoader(fn func() (RESTMapper, runtime.ObjectTyper, error)) (RESTMapper, runtime.ObjectTyper) {
	obj := &lazyObject{loader: fn}
	return obj, obj
}
```

看到此，便明白了```mapper, typer := f.objectMappingFactory.Object()``` 该条语句的返回值mapper, typer 接收的是两个同样的变量obj。 两步不同类型的返回值声明(RESTMapper，runtime.ObjectTyper)为什么接收是同样类型变量呢？再深入看下去便得知：

1. RESTMapper，runtime.ObjectTyper 是两个接口类型，它们的定义分别位于

   k8s.io/apimachinery/pkg/api/meta/interfaces.go 中和k8s.io/apimachinery/pkg/runtime/interfaces.g中。

2. lazyObject 这个结构体对象实现了上述两个接口中的方法。

上述返回变量的方式算是go 语言的特性了。看完上边的代码，需要记住的是

1. mapper, typer 的实际是一个lazyObject 结构体实例。
2. lazyObject 结构体实例 的loader 属性是方法```func (f *ring1Factory) objectLoader() (meta.RESTMapper, runtime.ObjectTyper, error)``` 的引用。

回到Newbuilder() 方法，再看clientMapperFunc，unstructuredClientMapperFunc 是如何得到返回值的。看下相关的代码实现：

```
type ClientMapper interface {
	ClientForMapping(mapping *meta.RESTMapping) (RESTClient, error)
}
// ClientMapperFunc implements ClientMapper for a function
type ClientMapperFunc func(mapping *meta.RESTMapping) (RESTClient, error)

func (f ClientMapperFunc) ClientForMapping(mapping *meta.RESTMapping) (RESTClient, error) {
	return f(mapping)
}
```

以上代码就是实现了接口ClientMapper。返回值```f(mapping)```  代表了一个传入实际参数后函数 f 在内存的地址引用。

根据传入ClientForMapping的参数值，得知 

1. clientMapperFunc，unstructuredClientMapperFunc 是两个函数，其函数类型定义是：type ClientMapperFunc func(mapping *meta.RESTMapping) (RESTClient, error)
2. clientMapperFunc 接收的函数是：f.objectMappingFactory.ClientForMapping
3. unstructuredClientMapperFunc 接收的函数是：f.objectMappingFactory.UnstructuredClientForMapping

最终，Newbuilder() 方法内调用resource.NewBuilder()  包装一个Builder 结构体实例作为返回值。通过上述分析，实例化过程直接表达如下：

```
resource.NewBuilder(
   &Builder{
	 internal: &resource.Mapper{
	    RESTMapper:   mapper,  // lazyObject 结构体实例
		ObjectTyper:  typer,   // lazyObject 结构体实例
	     //f.objectMappingFactory.ClientForMapping()方法
		ClientMapper: clientMapperFunc,  
		//Decoder的赋值过程函数调用太深。。。想吐
		Decoder:      f.clientAccessFactory.Decoder(true),
	 },
		           
	 unstructured: &resource.Mapper{
		RESTMapper:   mapper,  // lazyObject 结构体实例
		ObjectTyper:  typer,   // lazyObject 结构体实例	
		//f.objectMappingFactory.UnstructuredClientForMapping() 方法
		ClientMapper: unstructuredClientMapperFunc,  
		/* UnstructuredJSONScheme结构体，有用Decode、Encode、decode、encode 等多个实例方法
		结构体具体声明位于k8s.io/apimachinery/pkg/apis/meta/v1/unstructed.go */
		Decoder:      unstructured.UnstructuredJSONScheme 
		},
        
	 categoryExpander: categoryExpander,
	 requireObject:    true,
	}
）
```

需要记住的是： Builder 结构体实例中的属性internal, unstructured 均是一个 resource.Mapper 结构体。

至此，我们大概分析了RunCreate()函数中调用f.NewBuilder() 初始化一个Builder 实例的过程，并且上边的初始化过程只是对结构体部分属性的初始化，接下来就是调用Unstructured()等方法对builder 实例的属性值进行设置或修改 。

**Visitor 的构建**。

r 的构建跟builder属性相关，依次过一下处理Builder实例的函数流

1. Unstructured() : 对b.mapper 赋值，b.mapper = unstructured
2. Schema()： 对b.schema 赋值，b.schema = schema
3. ContinueOnError():  b.continueOnError 置为true。 意为遇到有错误的资源也不马上返回，跳过错误，继续处理下一个资源
4. NamespaceParam(cmdNamespace)：b.namespace = cmdNamespace
5. DefaultNamespace():  b.defaultNamespace 置为true
6. FilenameParam(enforceNamespace, &options.FilenameOptions) : 后边详细说
7. LabelSelectorParam(options.Selector)：对 b.labelSelector进行赋值
8. Flatten(): b.flatten  置为true,
9. Do()：后边详细说

再看visitorResult 中 对 r 的初始化，调用到FilenameParam() 方法时，第一次看到对visitor 的构建。

pkg/kubectl/resoure/builder.go

```
func (b *Builder) FilenameParam(enforceNamespace bool, filenameOptions *FilenameOptions) *Builder {
	recursive := filenameOptions.Recursive
	paths := filenameOptions.Filenames
	for _, s := range paths {
		switch {
		case s == "-":  // 使用本地yaml文件创建资
			b.Stdin()  // b.stream 置为true 更新b.paths 列表,添加FileVisitorForSTDIN 类型visitor 
			
		// 使用网络文件创建资源
		case strings.Index(s, "http://") == 0 || strings.Index(s, "https://") == 0:
			url, err := url.Parse(s)
			if err != nil {
				b.errs = append(b.errs, fmt.Errorf("the URL passed to filename %q is not   
				                valid: %v", s, err))
				continue
			}
			//更新b.paths 列表,添加URLVisitor类型visitor 
			b.URL(defaultHttpGetAttempts, url)
		default:
			if !recursive {
				b.singleItemImplied = true
			}
			
	        // 使用Declarative Management 方法创建对象走此路径
			b.Path(recursive, s)  
		}
	}

	if enforceNamespace { // 一般为true 
	    /*将b.requereNamepace 置为 true, 对缺省命令空间的资源使用cmdNamepace 或检查资源文件中   
	      namespace名字与命令行中指定的namespace 是否一致*/
		b.RequireNamespace()  
	}
	return b
}

```

注意到使用create -f xxx.json 命令时，会进入case 1 ，即调用b.Stdin() 方法。最后，b.paths 的结果是。

```
b.paths=[&FileVisitor{
		   Path: constSTDINstr,
		   *StreamVisitor: &StreamVisitor{  //输入流
		                      Reader: nil,
		                      Mapper: b.mapper (b.unstructured)
		                      Source: consSTDINstr,
		                      Schema: schema,
	                     },
	     }，
	   ]
```



遍历上边对Builder 处理方法，对visitor 有影响就是FilenameParam() 函数，接着就是到Do() 方法中重塑visitor。

Do() 方法具体实现如下：

pkg/kubectl/resoure/builder.go

```
func (b *Builder) Do() *Result {
	r := b.visitorResult()  // 第一次生成 result 实例，初始化r.visitor的值
	r.mapper = b.Mapper()
	if r.err != nil {
		return r    // 第一处return，出错时，从此处返回
	}
	if b.flatten {  // 默认为true, 在b.Flatten() 中赋值
		r.visitor = NewFlattenListVisitor(r.visitor, b.mapper) //第一次修改r.visitor 
	}
	helpers := []VisitorFunc{}
	if b.defaultNamespace {// 默认为true, 在b.DefaultNamespace() 赋值
		helpers = append(helpers, SetNamespace(b.namespace))
	}
	if b.requireNamespace { // 在FileFilenameParam() 赋值为true
	    /* b.namespace 即RunCreate() 中的cmdNamespace，cmdNamespace 的值为 kubectl 命令中-- 
	      namespace 参数的值*/
		helpers = append(helpers, RequireNamespace(b.namespace))
	}
	helpers = append(helpers, FilterNamespace)
	if b.requireObject {  // 默认为true，在builder 结构体初始化中赋值
		helpers = append(helpers, RetrieveLazy)
	}
	r.visitor = NewDecoratedVisitor(r.visitor, helpers...)  //  第二次修改r.visitor
	if b.continueOnError {
		r.visitor = ContinueOnErrorVisitor{r.visitor}
	}
	return r  //第二处return，一般从此处返回。
}
```



pkg/kubectl/resoure/visitor.go

```
func (b *Builder) visitorResult() *Result {
	if len(b.errs) > 0 {  
	    //返回一，错误返回，b.errs 为一列表结构，列表长度大于0，说明之前有错误发生，在此返回。
		return &Result{err: utilerrors.NewAggregate(b.errs)}
	}
	if b.selectAll { // create 命令中，b.selectAll 为false 
		selector := labels.Everything().String()
		b.labelSelector = &selector
	}

	/* create 命令进入此分支，例如 kubectl create -f xxx.yaml(或者URL/xxx.yaml)*/
	if len(b.paths) != 0 {
		return b.visitByPaths()  //返回二
	}

	// visit selectors
	if b.labelSelector != nil || b.fieldSelector != nil {
		return b.visitBySelector() // 返回三
	}

	// visit items specified by resource and name
	if len(b.resourceTuples) != 0 {
		return b.visitByResource()
	}

	// get 某一个指定资源对象时，进入此分支
	if len(b.names) != 0 {
		return b.visitByName()  // 返回四
	}

	if len(b.resources) != 0 {
		return &Result{err: fmt.Errorf("resource(s) were provided, but no name, label selector, or --all flag specified")}
	}
	return &Result{err: missingResourceError}
}
```



进入create 命令该走的分支，继续看 visitByPaths() 函数

pkg/kubectl/resoure/visitor.go

```
func (b *Builder) visitByPaths() *Result {
	result := &Result{
		singleItemImplied:  !b.dir && !b.stream && len(b.paths) == 1, // 结果为False
		targetsSingleItems: true,
	}

    ...
    /*错误处理代码*/
    ...

	var visitors Visitor
	if b.continueOnError { // 一般为true
	// EagerVisitorList 顾名思义就是(跳过错误)尽可能处理多的处理资源对象
		visitors = EagerVisitorList(b.paths)// b.paths 被包装为 EagerVisitorList 类型
	} else {
		visitors = VisitorList(b.paths)  // 遇到错误就返回
	}

	// only items from disk can be refetched
	if b.latest { // RunCreate() 中 b.latest = false
		// must flatten lists prior to fetching
		if b.flatten {
			visitors = NewFlattenListVisitor(visitors, b.mapper)
		}
		// must set namespace prior to fetching
		if b.defaultNamespace {
			visitors = NewDecoratedVisitor(visitors, SetNamespace(b.namespace))
		}
		visitors = NewDecoratedVisitor(visitors, RetrieveLatest)
	}
	if b.labelSelector != nil {  //RunCreate() 中，b.labelSelector 有可能能是true
		selector, err := labels.Parse(*b.labelSelector)
		if err != nil {
			return result.withError(fmt.Errorf("the provided selector %q is not valid: %v", b.labelSelector, err))
		}
		// 假定create命令进入此路径，visitor 被包装为一个FilteredVisitor 类型
		visitors = NewFilteredVisitor(visitors, FilterByLabelSelector(selector))
	}
	result.visitor = visitors  // 到此，r.visitor  为FilteredVisitor 类型
	result.sources = b.paths
	return result  // 返回result 到Do() 方法 
}
```



回到Do() 方法，r.visitor 又两次被修改类型。这就是执行create 命令时Do() 方法中 b.visitorResult() 的执行过程。梳理visitor 的变化过程大致如下：

![visit transition](..\image\visit transition.png)

至此，r 及visitor 实例的生成结束。下来又该返回到RunCreate() 函数中，执行r.visitor.Visit() 方法了。

