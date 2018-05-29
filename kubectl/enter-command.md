# 进入kubectl 命令

k8s.io/kubernetes/cmd/kubectl/kubectl.go

```
func main() {
	if err := app.Run(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
	os.Exit(0)
}
```

在Run() 方法中，创建和执行一个kubectl 命令， 这个命令使用cobra 框架实现。Cobra是一个go 语言实现的命令接口框架，个人理解其类似于OpenStack 命令实现中用到的Cliff 。

Run() 里继续调用NewKubectlCommand()   与 execute() 方法。

k8s.io/kubernetes/cmd/kubectl/app/kubectl.go

```
func Run() error {
	logs.InitLogs()
	defer logs.FlushLogs()

	cmd := cmd.NewKubectlCommand(cmdutil.NewFactory(nil), os.Stdin, os.Stdout, os.Stderr)
	return cmd.Execute()
}
```



在NewKubectlCommand()  *[k8s.io/kubernetes/pkg/kubectl/cmd/cmd.go ]中* ，通过直接或间接调用cmd.AddCommand() 完成子命令的添加，生成kubectl 命令集。 所有的命令都实现了 cobra.Command接口，该接口具有的属性及方法如下：

github.com/spf13/cobra/command.go 

```
type Command struct {
	name string
	Use string
	Short string
	Long string
	Example string
	parent *Command
	...
    PersistentPreRun()
	PreRun()
    Run()
    PostRun()
	PersistentPostRun()
	...
}

func (c *Command) Execute() { } 
func (c *Command) ExecuteC() (cmd *Command, err error) { }
func (c *Command) execute(a []string) (err error) { }

//AddCommad 方法中实现添加子命令，及对子命令的parent属性赋值。
func (c *Command) AddCommand(cmds ...*Command) { }
...
```

kubectl 相关命令实现该接口（属性与方法）时，重点关注的是Run() 方法，Run() 方法是更具体的任务执行的方法。例如，在NewKubectlCommand() 实现了kubectl 的顶层命令```kubectl xxx``` ，在该顶层方法中抽象Run() 用runHelp()  进行实例化;在其子命令```kubectl create -f xxx ```的方法中，抽象Run() 在NewCmdCreate(）方法中实现。

那么这些Run( ) 方法是何时被调用的呢？回到cmd.Execute()，看一下Execute( ) 方法的行为。Execute() 方法的大致执行流程是 Execute() -> ExecuteC() ->execute()，走到execute 方法，看其实现：

github.com/spf13/cobra/command.go 

```
 func (c *Command) execute(a []string) (err error) {
	...
	c.initHelpFlag()
    ...

	c.preRun()
	... 
	if c.RunE != nil {
		if err := c.RunE(c, argWoFlags); err != nil {
			return err
		}
	} else {
		c.Run(c, argWoFlags)
	}

	...
	return nil
}
```

可以看到，Command 接口中的各种 Run方法在execute都得到了调用。



下面以```kubectl create -f  filename ```命令执行开始，看一下pod 创建的执行流程。

 首先，看一下 create 命令的Run() 方法实现。

k8s.io/kubernetes/pkg/kubectl/cmd/create.go

```
func NewCmdCreate(f cmdutil.Factory, out, errOut io.Writer) *cobra.Command {
	...
	cmd := &cobra.Command{
	    ...
		Run: func(cmd *cobra.Command, args []string) {
			if cmdutil.IsFilenameSliceEmpty(options.FilenameOptions.Filenames) {
				defaultRunFunc := cmdutil.DefaultSubCommandRun(errOut)
				defaultRunFunc(cmd, args)
				return
			}
			cmdutil.CheckErr(options.ValidateArgs(cmd, args))
			cmdutil.CheckErr(RunCreate(f, cmd, out, errOut, &options))
		},
	}
	...
}	
```

再进一步，create命令中执行任务的是 RunCreate() 方法。首先需要明白参数 f 的来源。

向上回溯，发现参数f 是在较顶层的```cmd.NewKubectlCommand()```方法参数中生成的，

即 f = cmdutil.NewFactory(nil)。进入NewFactory() 方法，看其实现



k8s.io/kubernetes/pkg/kubectl/cmd/util/factory.go

```
func NewFactory(optionalClientConfig clientcmd.ClientConfig) Factory {
	clientAccessFactory := NewClientAccessFactory(optionalClientConfig)
	objectMappingFactory := NewObjectMappingFactory(clientAccessFactory)
	builderFactory := NewBuilderFactory(clientAccessFactory, objectMappingFactory)

	return &factory{
		ClientAccessFactory:  clientAccessFactory,
		ObjectMappingFactory: objectMappingFactory,
		BuilderFactory:       builderFactory,
	}
}
```

虽然NewFactory() 的实现看起来简单，返回的是一个初始化的factory 结构体实例。factory结构体声明时只有三个变量，每一个都是一个接口名，每个接口下都有很多抽象的方法。所以，在实例化这个factory结构体的时候，实际上是对几十个抽象方法的实现。具体的：

1. clientAccessFactory变量是一个 ring0Factory 结构体实例。该结构体中，实现了接口ClientAccessFactory中定义的所有方法。*[k8s.io/kubernetes/pkg/kubectl/cmd/util/factory_client_access.go]*
2. objectMappingFactory 变量是一个ring1Factory 结构体实例。*[util/factory_object_mapping.go]*
3. builderFactory 变量是一个ring2Factory 结构体实例。*[util/factory_builder.go]*

```
discoveryFactory :=  &discoveryFactory{clientConfig: clientConfig}

clientCache := NewClientCache(clientConfig, discoveryFactory)

clientAccessFactory := &ring0Factory{
		flags:            flags,
		clientConfig:     clientConfig,
		discoveryFactory: discoveryFactory
		clientCache:      clientCache,
	}
	
objectMappingFactory := &ring1Factory{
		clientAccessFactory: clientAccessFactory, // 上边实例化的clientAccessFactory
	}
	
builderFactory:=  &ring2Factory{
		clientAccessFactory:  clientAccessFactory,
		objectMappingFactory: objectMappingFactory,
	}
```

ring1Factory,  ring2Factory 结构体的实例化都赖于ring0Factory结构体的实例clientAccessFactory 。所以，搞定clientAccessFactory 实例的初始化

 弄清了参数 f 的来历后，再返回到RunCreate() 方法，看其实现。

pkg/kubectl/cmd/create.go

```
func RunCreate(f cmdutil.Factory, cmd *cobra.Command, out, errOut io.Writer, options *CreateOptions) error {
	if len(options.Raw) > 0 {...}
	...
	
	/*cmdutil.GetFlagBool(cmd, "validate") 默认为true
	  schema 为Schema 列表，包含了对资源文件的验证方法列表*/
	schema, err := f.Validator(cmdutil.GetFlagBool(cmd, "validate")) 
	
	/*cmdNamepace 接收 命令行中指定的namepace 值,enforceNamespace 一般为true */ 
	cmdNamespace, enforceNamespace, err := f.DefaultNamespace() 
	
	r := f.NewBuilder().
		Unstructured().
		Schema(schema).
		ContinueOnError().
		NamespaceParam(cmdNamespace).DefaultNamespace().
		FilenameParam(enforceNamespace, &options.FilenameOptions).
		LabelSelectorParam(options.Selector).
		Flatten().
		Do()
    ...
	err = r.Visit(func(info *resource.Info, err error) error {...}
}
```

主要看f.NewBuilder() 和r.Visit()。f.NewBuilder() 返回一个Builder 结构体 *[pkg/kubectl/resource/builder.go]* 实例。f.NewBuilder() 后边调用的方法很多，Unstructured、Schema、ContinueOnError、NamespaceParam、FilenameParam、LabelSelectorParam和Flatten都是Builder结构的方法，并返回Builder 结构的指针。执行这些方法类似一个流水线工作，每个方法执行一些对Builder结构体实例的修改，并且将这个实例返回给中线上的下一个方法来执行其他修改，最后实例调用Do() 方法，这些方法的实现都在```pkg/kubectl/resource/result.go```  文件中。Do() 方法返回值 r 的是一个Result 结构体实例 *[pkg/kubectl/resource/result.go]*。

 接着对象 r 调用Visit()方法 。传入Visit()的参数一个VisitorFunc 类型的匿名函数 ，记匿名函数名称为 fn_ins0, 该函数真正执行的地方将在 **<u>Visit 方法执行</u>** 一节 分析。先看到Visit()的实现：

pkg/kubectl/resource/result.go
```
func (r *Result) Visit(fn VisitorFunc) error {
    if r.err != nil {
		return r.err
	}
	err := r.visitor.Visit(fn)
	return utilerrors.FilterOut(err, r.ignoreErrors...)
}
```

对象r的属性visitor 为 Visitor 接口类型，Visitor 该接口的声明为：

pkg/kubectl/resource/visitor.go
```
type Visitor interface {
	Visit(VisitorFunc) error
}
```

代码中使用了多态的方式实现了该接口（主要集中在pkg/kubectl/resource/visitor.go 和 pkg/kubectl/resource/selector.go文件中）。此处具体调用的是哪个Visit() 方法，需要回到Do() 方法中看Result 实例的生成及属性visitor 被赋予的具体类型。

下文将分析Builder 结构体实例 及 visitor 属性的形成。






