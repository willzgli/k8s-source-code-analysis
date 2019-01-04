##                            volumeManager 工作原理分析

上文说到，在kubelet 服务启动过程中，在通过启动一个协程 kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)， 来运行volumeManager 组件的服务。

首先看一下与 volumeManager 负责业务的相关数据结构

kuberentes/pkg/kubelet/volumemanager/volume_manager.go

```
type volumeManager struct {
	kubeClient clientset.Interface
	volumePluginMgr *volume.VolumePluginMgr
	desiredStateOfWorld cache.DesiredStateOfWorld
	actualStateOfWorld cache.ActualStateOfWorld
	operationExecutor operationexecutor.OperationExecutor
	reconciler reconciler.Reconciler
	desiredStateOfWorldPopulator populator.DesiredStateOfWorldPopulator
}
```

在volumeManager 结构体中，有两个重要的变量desiredStateOfWorld， actualStateOfWorld，它们描述了volume 的不同状态：

- 2 个变量在定义的时候，都声明为interface 类型，但实例化的时候，都被赋予了一个结构体类型。具体的，desiredStateOfWorld  为desiredStateOfWorld 结构体类型， actualStateOfWorld 为actualStateOfWorld 结构体类型，它们都在`pkg/kubelet/volumemanager/cache` 下定义。

- desiredStateOfWorld：预期状态，该状态在pod 的yaml 文件中定义，当成功提交yaml文件后，预期状态就已经确定。

- actualStateOfWorld：实际状态，实际使用中，volume 的使用状态。实际状态是kubelet的后台线程监控的结果

  ​

volumeManager 正是基于这两个不同的状态，实现对本节点volume的管理。结构体中operationExecutor，reconciler，desiredStateOfWorldPopulator属性实质是三个接口，它们的作用已在kubelet 启动一节中简单描述，接口中定义的方法将在后边分析中用到时具体说明。volumeManager 结构体已在kubelet 启动时进行了实例化，并赋值给变量vm。

volumeManager定义了一个接口 VolumeManager，该接口作为kubelet进行数据卷管理的顶层方法入口，接口中声明的方法如下：

 kuberentes/pkg/kubelet/volumemanager/volume_manager.go

```
type VolumeManager interface {
	Run(sourcesReady config.SourcesReady, stopCh <-chan struct{})
	WaitForAttachAndMount(pod *v1.Pod) error
	GetMountedVolumesForPod(podName types.UniquePodName) container.VolumeMap
	GetExtraSupplementalGroupsForPod(pod *v1.Pod) []int64
	GetVolumesInUse() []v1.UniqueVolumeName
	ReconcilerStatesHasBeenSynced() bool
	VolumeIsAttached(volumeName v1.UniqueVolumeName) bool
	MarkVolumesAsReportedInUse(volumesReportedAsInUse []v1.UniqueVolumeName)
}
```

该接口中声明的方法作为volumeManager 结构体的成员方法，由volumeManager 负责实现。

首先，跟随volumeManager协程启动， 进入Run() 方法：

 kuberentes/pkg/kubelet/volumemanager/volume_manager.go

```
func (vm *volumeManager) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
	defer runtime.HandleCrash()

	go vm.desiredStateOfWorldPopulator.Run(sourcesReady, stopCh)
	glog.V(2).Infof("The desired_state_of_world populator starts")

	glog.Infof("Starting Kubelet Volume Manager")
	go vm.reconciler.Run(stopCh)

	<-stopCh
	glog.Infof("Shutting down Kubelet Volume Manager")
}
```

在Run() 方法中，主要启动了两个协程，分别运行vm.desiredStateOfWorldPopulator.Run() 和vm.reconciler.Run()这两个方法,其中:

**desiredStateOfWorldPopulator.Run() **方法主要根据从apiserver同步到的pod信息，来构建或者更新desiredStateOfWorld;

**vm.reconciler.Run()**是预期状态和实际状态的协调者，主要是比较actualStateOfWorld和desiredStateOfWorld的差别，然后进行volume的创建、删除和修改，将实际状态调整成与预期状态。

#### desiredStateOfWorld 生成

上面说到desiredStateOfWorld 的生成主要由vm.desiredStateOfWorldPopulator.Run() 协程负责。 进入vm.desiredStateOfWorldPopulator.Run()  方法，主要看其调用的下一级方法 populatorLoopFunc()

kubernetes/pkg/kubelet/volumemanager/populator/desired_state_of_world.go

```
func (dswp *desiredStateOfWorldPopulator) populatorLoopFunc() func() {
	return func() {
		dswp.findAndAddNewPods()

		// findAndRemoveDeletedPods() calls out to the container runtime to
		// determine if the containers for a given pod are terminated. This is
		// an expensive operation, therefore we limit the rate that
		// findAndRemoveDeletedPods() is called independently of the main
		// populator loop.
		if time.Since(dswp.timeOfLastGetPodStatus) < dswp.getPodStatusRetryDuration {
			glog.V(5).Infof(
				"Skipping findAndRemoveDeletedPods(). Not permitted until %v (getPodStatusRetryDuration %v).",
				dswp.timeOfLastGetPodStatus.Add(dswp.getPodStatusRetryDuration),
				dswp.getPodStatusRetryDuration)

			return
		}

		dswp.findAndRemoveDeletedPods()
	}
}
```

在首先执行findAndAddNewPods，然后执行findAndRemoveDeletedPods，由于findAndRemoveDeletedPods代价比较高昂，因此会检查执行的间隔时间

findAndAddNewPods的通过podManager获取所有的pods，然后调用processPodVolumes去更新desiredStateOfWorld。

findAndRemoveDeletedPods会遍历所有的volumeToMount，首先会判断该Volume所属的Pod是否存在于podManager，如果存在，则忽略，说明不需要删除。如果不存在，调用dswp.kubeContainerRuntime.GetPods(false)抓取Pod信息，这里是调用kubeContainerRuntime的GetPods函数。因此获取的都是runningPods信息，即正在运行的Pod信息。由于一个volume可以属于多个Pod，而一个Pod可以包含多个container，每个container都可以使用volume，所以他要扫描该volume所属的Pod的container信息，确保没有container使用该volume，才会删除该volume。

通过以上两步，desiredStateOfWorld就构建出来了，这是理想的volume状态，这里并没有发生实际的volume的创建删除挂载卸载操作。实际的操作由reconciler.Run(sourcesReady, stopCh)完成。

#### actualStateOfWorld 调整

actualStateOfWorld 调整主要由vm.reconciler.run() 方法的负责。vm.reconciler 结构体的实例化过程已经在kubelet 服务启动的时候完成。reconciler 结构体中的operationExecutor 变量 同样为结构体，该结构体的定义位于kubernetes/pkg/volume/util/operationexectutor/operation_executor.go，结构体中实现了AttachVolume, DetachVolume, MountVolume, UnmountVolume, UnmountDevice  等方法，负责对volume 的attach/detach/mount/unmount 操作，并更新actualStateOfWorld 的状态 ，简单对上述方法做一个说明：

- AttachVolume：调用volumeAttacher.Attach() 方法，将volume 作为一个块存储设备添加的到节点的/dev 目录下。

- DetachVolume：与AttachVolume 作用相反

- MountVolume： 调用volumeAttacher.MountDevice() ，将volume 挂载到节点的一个全局目录；再调用volumeMounter.SetUp()   将这个全局目录挂载到pod的卷目录。如此，就能将同一个volume挂载到多个pod，实现volume共享。

- UnmountVolume：与volumeMounter.SetUp()  作用相反，调用volumeUnmounter.TearDown()。

- UnmountDevice：与volumeAttacher.MountDevice()  作用相反，调用volumeDetacher.UnmountDevice() 方法

  ​

进入vm.reconciler.Run() 方法, 重点看其调用的reconciliationLoopFunc() 方法

kubernetes/pkg/kubelet/volumemanager/reconciler/reconciler.go

```
func (rc *reconciler) reconciliationLoopFunc(populatorHasAddedPods bool) func() {
	return func() {
		rc.reconcile()
		if populatorHasAddedPods && time.Since(rc.timeOfLastSync) > rc.syncDuration {
			glog.V(5).Infof("Desired state of world has been populated with pods, starting reconstruct state function")
			rc.sync()
		}
	}
}
```

先看rc.reconcile 方法的实现。该方法的实现代码较长，拆分代码，依次说明。

总体上看，该方法由3个大的for循环框架构成

```
//
for _, mountedVolume := range rc.actualStateOfWorld.GetMountedVolumes() { //for 循环1
...
}

for _, volumeToMount := range rc.desiredStateOfWorld.GetVolumesToMount() { //for 循环2
...
}

for for _, attachedVolume := range rc.actualStateOfWorld.GetUnmountedVolumes() { //for 循环3
...
}
```

总体思想：

volume和pod的预期状态不存在绑定关系，则detach volume，并对pod和volume执行unmount操作

1.  遍历实际状态中已挂载的volumes ，和预期状态比较，对不再和某个pod 存在绑定关系的volume 进行UnmountVolume  操作，以便在下一步中可能将要被本主机上面其它的容器所使用。
2.  对预期状态中的volume 和node 进行AttachVolume，MountVolume 操作。如for 循环2 实现
3.  遍历实际状态中已卸载的volumes，和预期状态比较，  确保应当解挂的存储解挂，这个上面第一个的区别是，上面只是从pod 的目录中进行unmount，下面则是包括了从本节点目录下unmount,detach。执行的是UnmountDevice 操作 和DetachVolume操作。

再细看每一个for 循环的实现

for 循环1：

```
for _, mountedVolume := range rc.actualStateOfWorld.GetMountedVolumes() { 
		if !rc.desiredStateOfWorld.PodExistsInVolume(mountedVolume.PodName, mountedVolume.VolumeName) {
			volumeHandler, err := operationexecutor.NewVolumeHandler(mountedVolume.VolumeSpec, rc.operationExecutor)
			if err != nil {
				glog.Errorf(mountedVolume.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.NewVolumeHandler for UnmountVolume failed"), err).Error())
				continue
			}
			err = volumeHandler.UnmountVolumeHandler(mountedVolume.MountedVolume, rc.actualStateOfWorld)
			if err != nil &&
				!nestedpendingoperations.IsAlreadyExists(err) &&
				!exponentialbackoff.IsExponentialBackoff(err) {
				// Ignore nestedpendingoperations.IsAlreadyExists and exponentialbackoff.IsExponentialBackoff errors, they are expected.
				// Log all other errors.
				glog.Errorf(mountedVolume.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.UnmountVolume failed (controllerAttachDetachEnabled %v)", rc.controllerAttachDetachEnabled), err).Error())
			}
			if err == nil {
				glog.Infof(mountedVolume.GenerateMsgDetailed("operationExecutor.UnmountVolume started", ""))
			}
		}
	}
```

大致流程如下：

1. 使用GetMountedVolumes() 获取已经挂载的volume 及pod 信息。返回一个列表，元素类型为：

   MountedVolume 结构体，该结构体定义在kubernetes/pkg/volume/util/operation_exectutor.go文件中。

   一个volume可能挂载到多个pod,该方法会分别返回volume与每一个pod 分别组成的  MountedVolume 结构体。

2. 根据  desiredStateOfWorld 状态的描述，遍历mountedVolume 列表，检查每一个MountedVolume 中的pod 与volume 是否还具有绑定关系。如果不在具有绑定关系，生成 volumeHandler ，处理解绑任务。

3. 在生成volumeHandler 的时候，会根据volumeMode(FileSystem or Block Device)  生成FilesystemVolumeHandler 或者BlockVolumeHandler。（默认为FilesystemVolumeHandler)

4.   调用volumeHandler.UnmountVolumeHandler() 方法，该方法最终会调用上面提到的UnmountVolume方法。




for 循环2 ：

```
for _, volumeToMount := range rc.desiredStateOfWorld.GetVolumesToMount() {
		volMounted, devicePath, err := rc.actualStateOfWorld.PodExistsInVolume(...)
		volumeToMount.DevicePath = devicePath
		if cache.IsVolumeNotAttachedError(err) {
		      // 如果Attach/Detach Controller 打开或者卷插件实现了volume.Attacher 接口,
		         等待Attach/Detach Controller 挂载Volume 
			if rc.controllerAttachDetachEnabled || !volumeToMount.PluginIsAttachable {
				...
				rr := rc.operationExecutor.VerifyControllerAttachedVolume(...)
				...
			} else {
				...
				// 如果Attach/Detach Controller 未打开,kubelet 服务自己挂载Volume
				err := rc.operationExecutor.AttachVolume(volumeToAttach, rc.actualStateOfWorld)
				...
			}
		} else if !volMounted || cache.IsRemountRequiredError(err) {
			// volume 已经attach, 需要重新mount
		    ...
			volumeHandler, err := operationexecutor.NewVolumeHandler(...)
		    ...
			err = volumeHandler.MountVolumeHandler(...)
		}	
	}
```



for 循环3：

```
	// Ensure devices that should be detached/unmounted are detached/unmounted.
	for _, attachedVolume := range rc.actualStateOfWorld.GetUnmountedVolumes() {
	
		if !rc.desiredStateOfWorld.VolumeExists(attachedVolume.VolumeName) &&
			!rc.operationExecutor.IsOperationPending(attachedVolume.VolumeName, nestedpendingoperations.EmptyUniquePodName) {
			if attachedVolume.GloballyMounted {
				volumeHandler, err := operationexecutor.NewVolumeHandler(attachedVolume.VolumeSpec, rc.operationExecutor)
				if err != nil {
					glog.Errorf(attachedVolume.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.NewVolumeHandler for UnmountDevice failed"), err).Error())
					continue
				}
				err = volumeHandler.UnmountDeviceHandler(attachedVolume.AttachedVolume, rc.actualStateOfWorld, rc.mounter)
				if err != nil &&
					!nestedpendingoperations.IsAlreadyExists(err) &&
					!exponentialbackoff.IsExponentialBackoff(err) {
					// Ignore nestedpendingoperations.IsAlreadyExists and exponentialbackoff.IsExponentialBackoff errors, they are expected.
					// Log all other errors.
					glog.Errorf(attachedVolume.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.UnmountDevice failed (controllerAttachDetachEnabled %v)", rc.controllerAttachDetachEnabled), err).Error())
				}
				if err == nil {
					glog.Infof(attachedVolume.GenerateMsgDetailed("operationExecutor.UnmountDevice started", ""))
				}
			} else {
				// Volume is attached to node, detach it
				// Kubelet not responsible for detaching or this volume has a non-attachable volume plugin.
				if rc.controllerAttachDetachEnabled || !attachedVolume.PluginIsAttachable {
					rc.actualStateOfWorld.MarkVolumeAsDetached(attachedVolume.VolumeName, attachedVolume.NodeName)
					glog.Infof(attachedVolume.GenerateMsgDetailed("Volume detached", fmt.Sprintf("DevicePath %q", attachedVolume.DevicePath)))
				} else {
					// Only detach if kubelet detach is enabled
					glog.V(12).Infof(attachedVolume.GenerateMsgDetailed("Starting operationExecutor.DetachVolume", ""))
					err := rc.operationExecutor.DetachVolume(
						attachedVolume.AttachedVolume, false /* verifySafeToDetach */, rc.actualStateOfWorld)
					if err != nil &&
						!nestedpendingoperations.IsAlreadyExists(err) &&
						!exponentialbackoff.IsExponentialBackoff(err) {
						// Ignore nestedpendingoperations.IsAlreadyExists && exponentialbackoff.IsExponentialBackoff errors, they are expected.
						// Log all other errors.
						glog.Errorf(attachedVolume.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.DetachVolume failed (controllerAttachDetachEnabled %v)", rc.controllerAttachDetachEnabled), err).Error())
					}
					if err == nil {
						glog.Infof(attachedVolume.GenerateMsgDetailed("operationExecutor.DetachVolume started", ""))
					}
				}
			}
		}
	}
```





http://www.voidcn.com/article/p-aqwpisit-bqo.html