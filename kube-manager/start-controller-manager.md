# **启动controller-manager **

Controller Manager: 集群内部的管理控制中心，主要目的是实现集群的故障检测和自动恢复工作 。

同样，controller manager 的启动在对应代码模块的main() 函数中。

kubernetes/cmd/kube-controller-manager/controller-manager.go 

```
func main(){
   s := options.NewCMServer() // 根据配置文件生成CMServer 结构体变量
   s.AddFlags(pflag.CommandLine, app.KnownControllers(),       
              app.ControllersDisabledByDefault.List())
   flag.InitFlags()
   logs.InitLogs()
   defer logs.FlushLogs() 
   verflag.PrintAndExitIfRequested()
   if err := app.Run(s); err != nil { // 启动各个controller
	   fmt.Fprintf(os.Stderr, "%v\n", err)
	   os.Exit(1)
   }
}
```

 

```
func NewCMServer() *CMServer {
	gcIgnoredResources := make([]componentconfig.GroupResource, 0, len(garbagecollector.DefaultIgnoredResources()))
	for r := range garbagecollector.DefaultIgnoredResources() {
		gcIgnoredResources = append(gcIgnoredResources, componentconfig.GroupResource{Group: r.Group, Resource: r.Resource})
	} // 

	s := CMServer{
		// Part of these default values also present in 'cmd/cloud-controller-manager/app/options/options.go'.
		// Please keep them in sync when doing update.
		KubeControllerManagerConfiguration: componentconfig.KubeControllerManagerConfiguration{
			Controllers:                   []string{"*"}, // 默认所有的controller
			Port:                           ports.ControllerManagerPort, // 默认端口10252
			Address:                                         "0.0.0.0",
			//Concurrentxxx: 每一个同步服务的并发数量
			ConcurrentEndpointSyncs:                         5
			...
			// xxxPeriod : 同步服务的运行间隔时间
			ServiceSyncPeriod:                               metav1.Duration{Duration: 5 * time.Minute},
			
			HorizontalPodAutoscalerUpscaleForbiddenWindow:   metav1.Duration{Duration: 3 * time.Minute},
			HorizontalPodAutoscalerDownscaleForbiddenWindow: metav1.Duration{Duration: 5 * time.Minute},
			HorizontalPodAutoscalerTolerance:                0.1,
			DeploymentControllerSyncPeriod:                  metav1.Duration{Duration: 30 * time.Second},
			MinResyncPeriod:                                 metav1.Duration{Duration: 12 * time.Hour},
			RegisterRetryCount:                              10,
			PodEvictionTimeout:                              metav1.Duration{Duration: 5 * time.Minute},
			NodeMonitorGracePeriod:                          metav1.Duration{Duration: 40 * time.Second},
			NodeStartupGracePeriod:                          metav1.Duration{Duration: 60 * time.Second},
			NodeMonitorPeriod:                               metav1.Duration{Duration: 5 * time.Second},
			ClusterName:                                     "kubernetes",
			NodeCIDRMaskSize:                                24,
			ConfigureCloudRoutes:                            true,
			TerminatedPodGCThreshold:                        12500,
			VolumeConfiguration: componentconfig.VolumeConfiguration{
				EnableHostPathProvisioning: false,
				EnableDynamicProvisioning:  true,
				PersistentVolumeRecyclerConfiguration: componentconfig.PersistentVolumeRecyclerConfiguration{
					MaximumRetry:             3,
					MinimumTimeoutNFS:        300,
					IncrementTimeoutNFS:      30,
					MinimumTimeoutHostPath:   60,
					IncrementTimeoutHostPath: 30,
				},
				FlexVolumePluginDir: "/usr/libexec/kubernetes/kubelet-plugins/volume/exec/",
			},
			ContentType:                           "application/vnd.kubernetes.protobuf",
			KubeAPIQPS:                            20.0,
			KubeAPIBurst:                          30,
			LeaderElection:                        leaderelectionconfig.DefaultLeaderElectionConfiguration(),
			ControllerStartInterval:               metav1.Duration{Duration: 0 * time.Second},
			EnableGarbageCollector:                true,
			ConcurrentGCSyncs:                     20,
			GCIgnoredResources:                    gcIgnoredResources,
			ClusterSigningCertFile:                DefaultClusterSigningCertFile,
			ClusterSigningKeyFile:                 DefaultClusterSigningKeyFile,
			ClusterSigningDuration:                metav1.Duration{Duration: helpers.OneYear},
			ReconcilerSyncLoopPeriod:              metav1.Duration{Duration: 60 * time.Second},
			EnableTaintManager:                    true,
			HorizontalPodAutoscalerUseRESTClients: true,
		},
	}
	s.LeaderElection.LeaderElect = true
	return &s
}
```





main() 函数中，主要的实现都在app.Run()方法中。以下是Run() 方法的实现

cmd/kube-controller-manager/app/controllermanager.go

```
func Run(s *options.CMServer) error {
    ...
	run := func(stop <-chan struct{}) {
		rootClientBuilder := controller.SimpleControllerClientBuilder{
			ClientConfig: kubeconfig,
		}
	   ...
			clientBuilder = controller.SAControllerClientBuilder{
				ClientConfig:         restclient.AnonymousClientConfig(kubeconfig),
				CoreClient:           kubeClient.CoreV1(),
				AuthenticationClient: kubeClient.Authentication(),
				Namespace:            "kube-system",
			}
		} 
		...
		ctx, err := CreateControllerContext(s, rootClientBuilder, clientBuilder, stop)
		if err != nil {
			glog.Fatalf("error building controller context: %v", err)
		}
		saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController

		if err := StartControllers(ctx, saTokenControllerInitFunc, NewControllerInitializers()); err != nil {
			glog.Fatalf("error starting controllers: %v", err)
		}

	...
	}

  ...
}
```

在Run() 方法中，通过调用StartControllers() 方法，实现了对卷管理及其他管理服务的初始化。在StartControllers()方法的参数中，controllers 参数实际接收的是NewControllerInitializers()方法的返回值， 该返回值为一个key-value 结构，返回值的 每个元素的value是初始化方法start* ，其中关于卷管理控制器的初始化方法为startAttachDetachController()。

此外，在Run方法中，生成了ctx 变量

​

NewControllerInitializers() 的代码实现如下：

​

cmd/kube-controller-manager/app/controllermanager.go

```
func NewControllerInitializers() map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	controllers["replicationcontroller"] = startReplicationController
	controllers["podgc"] = startPodGCController
	controllers["resourcequota"] = startResourceQuotaController
	controllers["namespace"] = startNamespaceController
	controllers["serviceaccount"] = startServiceAccountController
	controllers["garbagecollector"] = startGarbageCollectorController
	controllers["daemonset"] = startDaemonSetController
	controllers["job"] = startJobController
	controllers["deployment"] = startDeploymentController
	controllers["replicaset"] = startReplicaSetController
	controllers["horizontalpodautoscaling"] = startHPAController
	controllers["disruption"] = startDisruptionController
	controllers["statefulset"] = startStatefulSetController
	controllers["cronjob"] = startCronJobController
	controllers["csrsigning"] = startCSRSigningController
	controllers["csrapproving"] = startCSRApprovingController
	controllers["csrcleaner"] = startCSRCleanerController
	controllers["ttl"] = startTTLController
	controllers["bootstrapsigner"] = startBootstrapSignerController
	controllers["tokencleaner"] = startTokenCleanerController
	controllers["service"] = startServiceController
	controllers["node"] = startNodeController
	controllers["route"] = startRouteController
	controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
	controllers["attachdetach"] = startAttachDetachController
	controllers["persistentvolume-expander"] = startVolumeExpandController

	return controllers
}
```

再次返回到StartControllers()方法中，StartControllers() 代码的实现如下：
​

cmd/kube-controller-manager/app/controllermanager.go

```
func StartControllers(ctx ControllerContext, startSATokenController InitFunc, controllers map[string]InitFunc) error {
	...
	if _, err := startSATokenController(ctx); err != nil {
		return err
	}

    ...
	if ctx.Cloud != nil {
		ctx.Cloud.Initialize(ctx.ClientBuilder)
	}

    ...
	for controllerName, initFn := range controllers {
	   /* 是否需要初始化该controller */
		if !ctx.IsControllerEnabled(controllerName) {
			glog.Warningf("%q is disabled", controllerName)
			continue
		}

		time.Sleep(wait.Jitter(ctx.Options.ControllerStartInterval.Duration, ControllerStartJitter))
       started, err := initFn(ctx)  // 调用各个controller 的初始化方法start* 
      ...
	}

	return nil
}
```

上面提到controllers 为一个key-value结构的变量， 在 StartControllers() 方法中，`range controllers `遍历controllers 中的每一个元素，initFn 为元素的value 值，实际上为一个初始化方法。 本文主要分析卷管理的实现，以下是卷管理controller 的初始化方法startAttachDetachController()，

​