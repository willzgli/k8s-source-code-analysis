

# Visit 方法执行

上节分析了create 命令中，kubectl 中重要的结构体 builder  和 visitor 的形成。回到RunCreate()函数中，看一下r.Visit() 的运行。在Visit 的执行中，我们需要重点注意各种匿名函数和info 参数。

info 参数是一个结构体，具体声明如下

```
type Info struct {
	Client RESTClient
	Mapping *meta.RESTMapping
	Namespace string
	Name      string
	Source string
	Object runtime.Object
	ResourceVersion string
	Export bool
}
```

info 的具体赋值过程在后边的代码看到了再详说。

传入r.Visit()的变量是一个匿名函数，此处，记该匿名函数名字名字为fn_ins0。fn_ins0的实现如下：

```
fn_ins0 =: func (info *resource.Info, err error) error { // info 参数
	if err != nil {
		return err
	}
	if err := kubectl.CreateOrUpdateAnnotation(cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag), info, cmdutil.InternalVersionJSONEncoder()); err != nil {
		return cmdutil.AddSourceToErr("creating", info.Source, err)
	}

	if cmdutil.ShouldRecord(cmd, info) {
		if err := cmdutil.RecordChangeCause(info.Object, f.Command(cmd, false)); err != nil {
			return cmdutil.AddSourceToErr("creating", info.Source, err)
		}
	}

	if !dryRun {
		if err := createAndRefresh(info); err != nil {
		   return cmdutil.AddSourceToErr("creating", info.Source, err)
		}
	}

	count++

	shortOutput := output == "name"
	if len(output) > 0 && !shortOutput {
		return cmdutil.PrintObject(cmd,
		info.Object, o.Out)
	}

	cmdutil.PrintSuccess(shortOutput, o.Out,
	info.Object, dryRun, "created")
	return nil
}
```

进入 r.Visit() 方法 

```
func (r *Result) Visit(fn VisitorFunc) error {
	if r.err != nil {
		return r.err
	}
	//根据上节的分析，此处的Visit() 方法是func (v ContinueOnErrorVisitor) Visit(fn VisitorFunc)
	err := r.visitor.Visit(fn) // fn 即 fn_ins0
	return utilerrors.FilterOut(err, r.ignoreErrors...)
}
```



**进入ContinueOnErrorVisitor 类**

-----------------------------------------------------------------------------------------------------------------------------------------------------------

pkg/kubectl/resource/visitor.go

```
func (v ContinueOnErrorVisitor) Visit(fn VisitorFunc) error {// fn 为fn_ins0
	errs := []error{}
	err := v.Visitor.Visit(fn_ins1) // Visitor为DecoratedVisitor 实例，参数为匿名函数，记为fn_ins1
	if err != nil {
		errs = append(errs, err)
	}
	if len(errs) == 1 {
		return errs[0]
	}
	return utilerrors.NewAggregate(errs)
}
```

匿名函数fn_ins1 的实现为：

```
fn_ins1=func(info *Info, err error) error { // info 参数传给fn_ins0
		if err != nil {
			errs = append(errs, err)
			return nil
		}
		if err := fn(info, nil); err != nil {  // 最上层的匿名函数fn_ins0
			errs = append(errs, err)
		}
		return nil
	}
```



**进入DecoratedVisitor类**

-----------------------------------------------------------------------------------------------------------------------------------------------------------

```
func (v DecoratedVisitor) Visit(fn VisitorFunc) error { // 接收fn_ins1
	return v.visitor.Visit(fn_ins2) // visitor 为 NewFlattenListVisitor
}
```

匿名函数fn_ins2 实现

```
fn_ins2=func(info *Info, err error) error { //info 参数传给fn_ins1
		if err != nil {
			return err
		}
		for i := range v.decorators { // v 即DecoratedVisitor，遍历helper 函数
			if err := v.decorators[i](info, nil); err != nil { // 使用helper 处理info 
				return err
			}
		}
		return fn(info, nil)  // fn 为fn_ins1
	}
```



**进入NewFlattenListVisitor类**

-----------------------------------------------------------------------------------------------------------------------------------------------------------

```
func (v FlattenListVisitor) Visit(fn VisitorFunc) error { //接受fn_ins2
	return v.Visitor.Visit(fn_ins3) // Visitor为EagerVisitorList 列表
}
```

匿名函数fn_ins3 的实现

```
fn_ins3=func(info *Info, err error) error { // 传入info ，info 在该函数中被处理一次
		if err != nil {
			return err
		}
		if info.Object == nil {
			return fn(info, nil)
		}
		// 从info.Ojbect 生成items ,item 也为 *Info 类型
		items, err := meta.ExtractList(info.Object)
		if err != nil {
			return fn(info, nil)
		}
		if errs := runtime.DecodeList(items, v.Mapper.Decoder); len(errs) > 0 {
			return utilerrors.NewAggregate(errs)
		}

		var preferredGVKs []schema.GroupVersionKind
		if info.Mapping != nil && !info.Mapping.GroupVersionKind.Empty() {
			preferredGVKs = append(preferredGVKs, info.Mapping.GroupVersionKind)
		}

		for i := range items { // 对每个Item 调用 fn_ins2
			item, err := v.InfoForObject(items[i], preferredGVKs)
			if err != nil {
				return err
			}
			if len(info.ResourceVersion) != 0 {
				item.ResourceVersion = info.ResourceVersion
			}
			if err := fn(item, nil); err != nil { // fn为fn_ins2,传入item.
				return err
			}
		}
		return nil
	}
```



**进入 Visitor为EagerVisitorList 类**

-----------------------------------------------------------------------------------------------------------------------------------------------------------

```
func (l EagerVisitorList) Visit(fn VisitorFunc) error { //接收fn_ins3
	errs := []error(nil) // 初始化一个error 列表
	for i := range l { //l 为一个列表，每一个元素都是FileVistor 类,遍历每一个FileVistor 类
		if err := l[i].Visit(fn_ins4); err != nil { //l[i]FileVistor 类，一个路径下可能有多个文件
			errs = append(errs, err)   // if 接收的err 为nil,该句不执行。
		}
	}
	return utilerrors.NewAggregate(errs)  // 返回错误列表
}
```

匿名函数fn_ins4

```
fn_ins4=func(info *Info, err error) error {  // info 已不再是fn_ins2 用到的那个info
			if err != nil {
				errs = append(errs, err) // 记录错误
				return nil
			}
			if err := fn(info, nil); err != nil { //fn 即 fn_ins3
				errs = append(errs, err)  // 记录错误
			}
			return nil  
		}
```

 在fn_ins4 看到返回值始终nil , 错误只是记录。



**进入 FileVisitor 类**

-----------------------------------------------------------------------------------------------------------------------------------------------------------

```
func (v *FileVisitor) Visit(fn VisitorFunc) error {// 接收fn_ins4
	var f *os.File  //文件句柄
	if v.Path == constSTDINstr {
		f = os.Stdin
	} else {
		var err error
		f, err = os.Open(v.Path)
		if err != nil {
			return err
		}
		defer f.Close()
	}

	// TODO: Consider adding a flag to force to UTF16, apparently some
	// Windows tools don't write the BOM
	utf16bom := unicode.BOMOverride(unicode.UTF8.NewDecoder())
	v.StreamVisitor.Reader = transform.NewReader(f, utf16bom) //初始化Reader

	return v.StreamVisitor.Visit(fn) // 继续调用Visit，fn 为fn_ins4
}
```



**进入StreamVisitor 类**

-----------------------------------------------------------------------------------------------------------------------------------------------------------

```
func (v *StreamVisitor) Visit(fn VisitorFunc) error { 
	d := yaml.NewYAMLOrJSONDecoder(v.Reader, 4096)
	for { //无限循环
		ext := runtime.RawExtension{}
		if err := d.Decode(&ext); err != nil {  //读数据
			if err == io.EOF { //正确返回
				return nil
			}
			return err
		}
		// TODO: This needs to be able to handle object in other encodings and schemas.
		ext.Raw = bytes.TrimSpace(ext.Raw) 
		if len(ext.Raw) == 0 || bytes.Equal(ext.Raw, []byte("null")) {
			continue
		}
		if err := ValidateSchema(ext.Raw, v.Schema); err != nil {
			return fmt.Errorf("error validating %q: %v", v.Source, err)
		}
		
		/*终于看见info 的生成了。InfoForData 是Mapper 结构体的方法，但在StreamVisitor中包含了 	  	
		  *Mapper 属性/
		info, err := v.InfoForData(ext.Raw, v.Source) 
		if err != nil {
			if fnErr := fn(info, err); fnErr != nil { 
				return fnErr
			}
			continue
		}
		if err := fn(info, nil); err != nil { // 执行fn_ins4
			return err
		}
	}
}
```



在StreamVisitor.Visit() 方法中调用了InfoForData() 生成了info 这个实例，接着**多次**回调了这些匿名函数，执行流程如下图所示（红色的箭头线便为一次执行过程）：

![callback funcs](..\image\call-back functions.png)

对info 的加工处理，主要在fn_ins3, fn_ins2, fn_ins0 中。

fn_ins3:

fn_ins2:

fn_ins0:   进入```createAndRefresh(info)```, 在该方法内生成了一个Helper结构体实例，调用该实例的Create() , 将数据发给server端。



对于info 的生成，下一节详细展开。