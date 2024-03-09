# koordlet源码分析
文章分析主要分为静态分析和 动态分析

## 一、分析环境

koordnator软件版本

```
v1.4.0
```

运行环境

```
ubuntu21.04
```

## 源码解析 

### 启动模块分析

koordlet 核心模块初始化在 cmd/koordlet/main.go 中

```
d, err := agent.NewDaemon(cfg)
```

在 pkg/koordlet/koordlet.go 中NewDaemon 里主要初始化了 以下几个模块:

这个函数用来获取当前linux 主机的一些支持信息
```
system.InitSupportConfigs()
-> initJiffies // 本质是使用 getconf CLK_TCK 获取时钟精度
-> initCgroupsVersion // 判断是不是cgroupv2版本，通过 stat /sys/fs/cgroup 获取
-> collectVersionInfo // 主机信息	待理解
```

initCgroupsVersion 核心逻辑:
```
isUnifiedOnce.Do(func() {
	var st unix.Statfs_t
	err := unix.Statfs(unifiedMountpoint, &st)
	if err != nil {
		if os.IsNotExist(err) && userns.RunningInUserNS() {
			// ignore the "not found" error if running in userns
			klog.ErrorS(err, "%s missing, assuming cgroup v1", unifiedMountpoint)
			isUnified = false
			return
		}
		panic(fmt.Sprintf("cannot statfs cgroup root: %s", err))
	}
	isUnified = st.Type == unix.CGROUP2_SUPER_MAGIC
})
```

初始化 k8s 的client
```
kubeClient := clientset.NewForConfigOrDie(config.KubeRestConf)
crdClient := clientsetbeta1.NewForConfigOrDie(config.KubeRestConf)
topologyClient := topologyclientset.NewForConfigOrDie(config.KubeRestConf)
schedulingClient := v1alpha1.NewForConfigOrDie(config.KubeRestConf)
```

初始化指标的cache
```
metricCache, err := metriccache.NewMetricCache(config.MetricCacheConf)
```

初始化cgroup formatter

```
cgroupDriver := system.GetCgroupDriver()
system.SetupCgroupPathFormatter(cgroupDriver)
```

初始化指标收集器
```
collectorService := metricsadvisor.NewMetricAdvisor(config.CollectorConf, statesInformer, metricCache)
```

初始化 evictVersion

```
evictVersion, err := util.FindSupportedEvictVersion(kubeClient)
```

初始化qosManager

```
qosManager := qosmanager.NewQOSManager(config.QOSManagerConf, scheme, kubeClient, crdClient, nodeName, statesInformer, metricCache, config.CollectorConf, evictVersion)
```

初始化koordlet runtimeproxy的 server 插口

```
runtimeHook, err := runtimehooks.NewRuntimeHook(statesInformer, config.RuntimeHookConf)
```

最后使用run 调用各个模块

```
func (d *daemon) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	klog.Infof("Starting daemon")

	// start resource executor cache
	d.executor.Run(stopCh)

	go func() {
		if err := d.metricCache.Run(stopCh); err != nil {
			klog.Fatal("Unable to run the metric cache: ", err)
		}
	}()

	// start states informer
	go func() {
		if err := d.statesInformer.Run(stopCh); err != nil {
			klog.Fatal("Unable to run the states informer: ", err)
		}
	}()
	// wait for metric advisor sync
	if !cache.WaitForCacheSync(stopCh, d.statesInformer.HasSynced) {
		klog.Fatal("time out waiting for states informer to sync")
	}

	// start metric advisor
	go func() {
		if err := d.metricAdvisor.Run(stopCh); err != nil {
			klog.Fatal("Unable to run the metric advisor: ", err)
		}
	}()
	// wait for metric advisor sync
	if !cache.WaitForCacheSync(stopCh, d.metricAdvisor.HasSynced) {
		klog.Fatal("time out waiting for metric advisor to sync")
	}

	// start predict server
	go func() {
		if err := d.predictServer.Setup(d.statesInformer, d.metricCache); err != nil {
			klog.Fatal("Unable to setup the predict server: ", err)
		}
		if err := d.predictServer.Run(stopCh); err != nil {
			klog.Fatal("Unable to run the predict server: ", err)
		}
	}()

	// start qos manager
	go func() {
		if err := d.qosManager.Run(stopCh); err != nil {
			klog.Fatal("Unable to run the qosManager: ", err)
		}
	}()

	go func() {
		if err := d.runtimeHook.Run(stopCh); err != nil {
			klog.Fatal("Unable to run the runtimeHook: ", err)
		}
	}()

	klog.Info("Start daemon successfully")
	<-stopCh
	klog.Info("Shutting down daemon")
}

```

### metricCache 模块分析

metricCache 主要提供两种存储能力，具体代码实现位于 pkg/koordlet/metriccache/metric_cache.go中。

 - 1、存储[tsdb](https://www.taosdata.com/tsdb)
 - 2、内存存储，存储在sync.Map中。

### prediction 模块分析

#### 1、PeakPredictServer

初始化predictServer,核心代码实现位于pkg/koordlet/prediction/predict_server.go

```
func NewPeakPredictServer(cfg *Config) PredictServer {
	return &peakPredictServer{
		cfg:          cfg,
		uidGenerator: &generator{},
		//models 里面存储了 单个pod 所有pod、系统CPU 内存使用情况
		models:       make(map[UIDType]*PredictModel),
		clock:        clock.RealClock{},
		hasSynced:    &atomic.Bool{},
		checkpointer: NewFileCheckpointer(cfg.CheckpointFilepath),
	}
}
```

peakPredictServer->Run 负责定时把数据写入peakPredictServer的models 属性中，用来在后续调用GetPrediction读取CPU内存的分位数数据的计算结果,peakPredictServer->Run 核心代码解析:

```
// 重置 models
unknownUIDs := p.restoreModels()
// 定时读取POD，全部POD 系统的CPU 内存的统计数据，写入models 中
go wait.Until(p.training, p.cfg.TrainingInterval, stopCh)
// 清除掉models 中过期统计
go wait.Until(p.gcModels, time.Minute, stopCh)
//检查节点数据
go wait.Until(p.doCheckpoint, time.Minute, stopCh)
```

GetPrediction 是peakPredictServer 给外部调用的接口，用来计算CPU 和 内存的分位数，现在支持P60、P90、P95、P98、max 等

```
func (p *peakPredictServer) GetPrediction(metric MetricDesc) (Result, error) {
	p.modelsLock.Lock()
	defer p.modelsLock.Unlock()
	model, ok := p.models[metric.UID]
	if !ok {
		return Result{}, fmt.Errorf("UID %v not found in predict server", metric.UID)
	}
	model.Lock.Lock()
	defer model.Lock.Unlock()
	//
	return Result{
		Data: map[string]v1.ResourceList{
			"p60": {
				v1.ResourceCPU:    *resource.NewMilliQuantity(int64(model.CPU.Percentile(0.6)*1000.0), resource.DecimalSI),
				v1.ResourceMemory: *resource.NewQuantity(int64(model.Memory.Percentile(0.6)), resource.BinarySI),
			},
			"p90": {
				v1.ResourceCPU:    *resource.NewMilliQuantity(int64(model.CPU.Percentile(0.9)*1000.0), resource.DecimalSI),
				v1.ResourceMemory: *resource.NewQuantity(int64(model.Memory.Percentile(0.9)), resource.BinarySI),
			},
			"p95": {
				v1.ResourceCPU:    *resource.NewMilliQuantity(int64(model.CPU.Percentile(0.95)*1000.0), resource.DecimalSI),
				v1.ResourceMemory: *resource.NewQuantity(int64(model.Memory.Percentile(0.95)), resource.BinarySI),
			},
			"p98": {
				v1.ResourceCPU:    *resource.NewMilliQuantity(int64(model.CPU.Percentile(0.98)*1000.0), resource.DecimalSI),
				v1.ResourceMemory: *resource.NewQuantity(int64(model.Memory.Percentile(0.98)), resource.BinarySI),
			},
			"max": {
				v1.ResourceCPU:    *resource.NewMilliQuantity(int64(model.CPU.Percentile(1.0)*1000.0), resource.DecimalSI),
				v1.ResourceMemory: *resource.NewQuantity(int64(model.Memory.Percentile(1.0)), resource.BinarySI),
			},
		},
	}, nil
}

```

#### 2、predictorFactory

predictorFactory 是一个关于峰值预测的模型，接口定义：

```

type Predictor interface {
	GetPredictorName() string
	AddPod(pod *v1.Pod) error
	GetResult() (v1.ResourceList, error)
}
```



大约有4个实例，分别是

```
1、minPredictor
2、emptyPredictor
3、podReclaimablePredictor
4、priorityReclaimablePredictor
```

这些工厂是用来度量可以回收多少内存和CPU的，回收的标准是 pod中各个子容器(包括init容器)的度量值总和 - pod 实际使用的值

##### (1) emptyPredictor

不需要关注。统一返回nil


##### (2) podReclaimablePredictor

我们在这里只介绍

addPod数据时候回判断pod 上是否有

```
koordinator.sh/priority-class
```

如果存在这个标签会继续执行逻辑，AddPod核心计算逻辑

```
// 获取pod的所有容器的request 总值
podRequests := util.GetPodRequest(pod, v1.ResourceCPU, v1.ResourceMemory)
	podCPURequest := podRequests[v1.ResourceCPU]
	podMemoryRequest := podRequests[v1.ResourceMemory]

	reclaimableCPUMilli := int64(0)
	reclaimableMemoryBytes := int64(0)

	// 计算安全边界
	ratioAfterSafetyMargin := float64(100+p.safetyMarginPercent) / 100
	// 计算可以回收的CPU值
	if p95CPU, ok := p95Resources[v1.ResourceCPU]; ok {
		peakCPU := util.MultiplyMilliQuant(p95CPU, ratioAfterSafetyMargin)
		reclaimableCPUMilli = podCPURequest.MilliValue() - peakCPU.MilliValue()
	}
	//计算可以回收的内存值
	if p98Memory, ok := p98Resources[v1.ResourceMemory]; ok {
		peakMemory := util.MultiplyQuant(p98Memory, ratioAfterSafetyMargin)
		reclaimableMemoryBytes = podMemoryRequest.Value() - peakMemory.Value()
	}

	// 记录到 reclaimable值里
	if reclaimableCPUMilli > 0 {
		cpu := p.reclaimable[v1.ResourceCPU]
		reclaimableCPU := resource.NewMilliQuantity(reclaimableCPUMilli, resource.DecimalSI)
		pu.Add(*reclaimableCPU)
		p.reclaimable[v1.ResourceCPU] = cpu
	}
	if reclaimableMemoryBytes > 0 {
		memory := p.reclaimable[v1.ResourceMemory]
		reclaimableMemory := resource.NewQuantity(reclaimableMemoryBytes, resource.BinarySI)
		memory.Add(*reclaimableMemory)
		p.reclaimable[v1.ResourceMemory] = memory
	}
```

GetResult 返回还可以回收多少内存和CPU资源

```

// GetResult returns the predicted resource list for the added pods.
func (p *podReclaimablePredictor) GetResult() (v1.ResourceList, error) {
	metrics.RecordNodePredictedResourceReclaimable(string(v1.ResourceCPU), metrics.UnitCore, p.GetPredictorName(), float64(p.reclaimable.Cpu().MilliValue())/1000)
	metrics.RecordNodePredictedResourceReclaimable(string(v1.ResourceMemory), metrics.UnitByte, p.GetPredictorName(), float64(p.reclaimable.Memory().Value()))
	return p.reclaimable, nil
}
```

##### (3) priorityReclaimablePredictor

podReclaimablePredictor 只会计算 pod上标注优先级为koord-prod 或者空的,和podReclaimablePredictor区别主要是和podReclaimablePredictor是计算pod维度的客户收预测量，而priorityReclaimablePredictor是product优先级级别的

```

func (n *priorityReclaimablePredictor) GetResult() (v1.ResourceList, error) {
	// get sys prediction
	sysResult, err := n.predictServer.GetPrediction(MetricDesc{UID: getNodeItemUID(SystemItemID)})
	if err != nil {
		return nil, fmt.Errorf("failed to get prediction of sys, err: %w", err)
	}
	sysResultForCPU := sysResult.Data["p95"]
	sysResultForMemory := sysResult.Data["p98"]
	reclaimPredict := v1.ResourceList{
		v1.ResourceCPU:    *sysResultForCPU.Cpu(),
		v1.ResourceMemory: *sysResultForMemory.Memory(),
	}

	// 遍历所有优先级，只找到优先级为koord-prod或者空的预测量
	// get reclaimable priority class prediction,
	for _, priorityClass := range extension.KnownPriorityClasses {
		if !n.priorityClassFilterFn(priorityClass) {
			continue
		}

		result, err := n.predictServer.GetPrediction(MetricDesc{UID: getNodeItemUID(string(priorityClass))})
		if err != nil {
			return nil, fmt.Errorf("failed to get prediction of priority %s, err: %s", priorityClass, err)
		}

		resultForCPU := result.Data["p95"]
		resultForMemory := result.Data["p98"]
		predictResource := v1.ResourceList{
			v1.ResourceCPU:    *resultForCPU.Cpu(),
			v1.ResourceMemory: *resultForMemory.Memory(),
		}
		reclaimPredict = quotav1.Add(reclaimPredict, predictResource)
	}

	// scale with the safety margin
	ratioAfterSafetyMargin := float64(100+n.safetyMarginPercent) / 100
	reclaimPredict = v1.ResourceList{
		v1.ResourceCPU:    util.MultiplyMilliQuant(*reclaimPredict.Cpu(), ratioAfterSafetyMargin),
		v1.ResourceMemory: util.MultiplyQuant(*reclaimPredict.Memory(), ratioAfterSafetyMargin),
	}

	// reclaimable[P] := max(request[P] - peak[P], 0)
	// 优先级下所有资源request用量 - 优先级下的实际用量
	reclaimable := quotav1.Max(quotav1.Subtract(n.reclaimRequest, reclaimPredict), util.NewZeroResourceList())
	metrics.RecordNodePredictedResourceReclaimable(string(v1.ResourceCPU), metrics.UnitCore, n.GetPredictorName(), float64(reclaimable.Cpu().MilliValue())/1000)
	metrics.RecordNodePredictedResourceReclaimable(string(v1.ResourceMemory), metrics.UnitByte, n.GetPredictorName(), float64(reclaimable.Memory().Value()))
	// 返回结果
	return reclaimable, nil
}
```

##### 4、minPredictor

minPredictor 是从上面所有预测器里找到最小预测值，并且返回

```
func (m *minPredictor) GetResult() (v1.ResourceList, error) {
	if len(m.predictors) <= 0 {
		return util.NewZeroResourceList(), nil
	}

	minimal, err := m.predictors[0].GetResult()
	if err != nil {
		return nil, fmt.Errorf("failed to get predictor %s result, error: %v", m.predictors[0].GetPredictorName(), err)
	}
	for i := 1; i < len(m.predictors); i++ {
		result, err := m.predictors[i].GetResult()
		if err != nil {
			return nil, fmt.Errorf("failed to get predictor %s result, error: %v", m.predictors[i].GetPredictorName(), err)
		}

		minimal = util.MinResourceList(minimal, result)
	}

	klog.V(6).Infof("minPredictor get result: %+v", minimal)
	metrics.RecordNodePredictedResourceReclaimable(string(v1.ResourceCPU), metrics.UnitCore, m.GetPredictorName(), float64(minimal.Cpu().MilliValue())/1000)
	metrics.RecordNodePredictedResourceReclaimable(string(v1.ResourceMemory), metrics.UnitByte, m.GetPredictorName(), float64(minimal.Memory().Value()))
	return minimal, nil
}
```

### statesInformer

States Informer 从 kube-apiserver 和 kubelet 同步节点和 Pod 状态，并将数据作为 static 类型保存到 Storage 中。与其他模块相比，该模块在开发迭代中应该保持相对稳定。



statesInformer 是和k8s api server 通讯的类，用来监听常用资源的变动并且同步到缓存中，如果是负载资源变动，会调用 predictorFactory 的AddPod 接口，同步峰值预测的CPU和内存的量值，支持的插件列表如下:

```
// NOTE: variables in this file can be overwritten for extension

var DefaultPluginRegistry = map[PluginName]informerPlugin{
	// 同步node信息
	nodeSLOInformerName:    NewNodeSLOInformer(),
	// pvc 信息
	pvcInformerName:        NewPVCInformer(),
	// 收集node 拓扑信息
	nodeTopoInformerName:   NewNodeTopoInformer(),
	// 
	nodeInformerName:       NewNodeInformer(),
	// 收集pod信息监听pod变化
	podsInformerName:       NewPodsInformer(),
	// 收集node信息变化
	nodeMetricInformerName: NewNodeMetricInformer(),
}
```

statesInformer->Run 运行所有插件

```
klog.V(2).Infof("starting informer plugins")
s.setupPlugins()
s.startPlugins(stopCh)
```

#### (1) nodeSLOInformer

 SLO(Service Level Objectives)来定义集群性能的衡量标准和集群性能要达到的目标。

nodeSLO 是 koordinator 定制的一个CRD资源，代码位置在：

```
apis/slo/v1alpha1/nodeslo_types.go
```

具体定义:

```
// +genclient
// +genclient:nonNamespaced
// +kubebuilder:object:root=true
// +kubebuilder:resource:scope=Cluster
// +kubebuilder:subresource:status

// NodeSLO is the Schema for the nodeslos API
type NodeSLO struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   NodeSLOSpec   `json:"spec,omitempty"`
	Status NodeSLOStatus `json:"status,omitempty"`
}

```

nodeSLOInformer 主要用来同步这个crd变化并且放到缓存里


```
func (s *nodeSLOInformer) Setup(ctx *PluginOption, state *PluginState) {
	s.nodeSLOInformer = newNodeSLOInformer(ctx.KoordClient, ctx.NodeName)
	s.nodeSLOInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		// 监听添加nodeslo
		AddFunc: func(obj interface{}) {
			nodeSLO, ok := obj.(*slov1alpha1.NodeSLO)
			if ok {
				// 同步到缓存中
				s.updateNodeSLOSpec(nodeSLO)
				klog.Infof("create NodeSLO %v", util.DumpJSON(nodeSLO))
			} else {
				klog.Errorf("node slo informer add func parse nodeSLO failed")
			}
		},
		// 监听更新同步到nodeslo
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldNodeSLO, oldOK := oldObj.(*slov1alpha1.NodeSLO)
			newNodeSLO, newOK := newObj.(*slov1alpha1.NodeSLO)
			if !oldOK || !newOK {
				klog.Errorf("unable to convert object to *slov1alpha1.NodeSLO, old %T, new %T", oldObj, newObj)
				return
			}
			if reflect.DeepEqfunc (s *nodeSLOInformer) Setup(ctx *PluginOption, state *PluginState) {
	s.nodeSLOInformer = newNodeSLOInformer(ctx.KoordClient, ctx.NodeName)
	s.nodeSLOInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			nodeSLO, ok := obj.(*slov1alpha1.NodeSLO)
			if ok {
				s.updateNodeSLOSpec(nodeSLO)
				klog.Infof("create NodeSLO %v", util.DumpJSON(nodeSLO))
			} else {
				klog.Errorf("node slo informer add func parse nodeSLO failed")
			}
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldNodeSLO, oldOK := oldObj.(*slov1alpha1.NodeSLO)
			newNodeSLO, newOK := newObj.(*slov1alpha1.NodeSLO)
			if !oldOK || !newOK {
				klog.Errorf("unable to convert object to *slov1alpha1.NodeSLO, old %T, new %T", oldObj, newObj)
				return
			}
			// 检查是否有变化
			if reflect.DeepEqual(oldNodeSLO.Spec, newNodeSLO.Spec) {
				klog.V(5).Infof("find NodeSLO spec %s has not changed", newNodeSLO.Name)
				return
			}
			klog.Infof("update NodeSLO spec %v", util.DumpJSON(newNodeSLO.Spec))
			// 更新同步到缓存中
			s.updateNodeSLOSpec(newNodeSLO)
		},
	})
	s.callbackRunner = state.callbackRunner
}ual(oldNodeSLO.Spec, newNodeSLO.Spec) {
				klog.V(5).Infof("find NodeSLO spec %s has not changed", newNodeSLO.Name)
				return
			}
			klog.Infof("update NodeSLO spec %v", util.DumpJSON(newNodeSLO.Spec))
			s.updateNodeSLOSpec(newNodeSLO)
		},
	})
	s.callbackRunner = state.callbackRunner
}
```

对外暴露接口获取配置

```

func (s *nodeSLOInformer) GetNodeSLO() *slov1alpha1.NodeSLO {
	s.nodeSLORWMutex.RLock()
	defer s.nodeSLORWMutex.RUnlock()
	return s.nodeSLO.DeepCopy()
}
```

#### (2)pvcInformer

pvcInformer 监听pvc 变化，同步pvc 信息到volumeNameMap中


```
func NewPVCInformer() *pvcInformer {
	return &pvcInformer{
		volumeNameMap: map[string]string{},
	}
}
```

对外暴露接口:

```
func (s *pvcInformer) GetVolumeName(pvcNamespace, pvcName string) string {
	s.pvcRWMutex.RLock()
	defer s.pvcRWMutex.RUnlock()
	return s.volumeNameMap[util.GetNamespacedName(pvcNamespace, pvcName)]
}

```

#### (3) nodeTopoInformer

这个informer 核心功能是定时调用nodeTopoInformer->reportNodeTopology,核心代码位于pkg/koordlet/statesinformer/impl/states_noderesourcetopology

#### (4) nodeInformer
监听node变化
```
func (s *nodeInformer) Setup(ctx *PluginOption, state *PluginState) {
	s.callbackRunner = state.callbackRunner

	s.nodeInformer = newNodeInformer(ctx.KubeClient, ctx.NodeName)
	s.nodeInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			node, ok := obj.(*corev1.Node)
			if ok {
				s.syncNode(node)
			} else {
				klog.Errorf("node informer add func parse Node failed, obj %T", obj)
			}
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldNode, oldOK := oldObj.(*corev1.Node)
			newNode, newOK := newObj.(*corev1.Node)
			if !oldOK || !newOK {
				klog.Errorf("unable to convert object to *corev1.Node, old %T, new %T", oldObj, newObj)
				return
			}
			if newNode.ResourceVersion == oldNode.ResourceVersion {
				klog.V(5).Infof("find node %s has not changed", newNode.Name)
				return
			}
			s.syncNode(newNode)
		},
	})
}
```

同步记录 node 申请的batch-cpu、batch-memory、mid-cpu、mid-memory

```
func recordNodeResources(node *corev1.Node) {
	if node == nil || node.Status.Allocatable == nil {
		klog.V(4).Infof("failed to record node resources metrics, node is invalid: %v", node)
		return
	}

	// record node allocatable of BatchCPU & BatchMemory
	batchCPU := node.Status.Allocatable.Name(apiext.BatchCPU, resource.DecimalSI)
	metrics.RecordNodeResourceAllocatable(string(apiext.BatchCPU), metrics.UnitInteger, float64(batchCPU.Value()))
	batchMemory := node.Status.Allocatable.Name(apiext.BatchMemory, resource.BinarySI)
	metrics.RecordNodeResourceAllocatable(string(apiext.BatchMemory), metrics.UnitByte, float64(batchMemory.Value()))

	// record node allocatable of MidCPU & MidMemory
	midCPU := node.Status.Allocatable.Name(apiext.MidCPU, resource.DecimalSI)
	metrics.RecordNodeResourceAllocatable(string(apiext.MidCPU), metrics.UnitInteger, float64(midCPU.Value()))
	midMemory := node.Status.Allocatable.Name(apiext.MidMemory, resource.BinarySI)
	metrics.RecordNodeResourceAllocatable(string(apiext.MidMemory), metrics.UnitByte, float64(midMemory.Value()))
}

```

#### (5) podInformer

当有pod变化时候，核心同步的函数是 syncPods，核心代码位于

```
pkg/koordlet/statesinformer/impl/states_pods.go
```

同步pod信息

```
func (s *podsInformer) syncPods() error {
	// 拉取pod列表
	podList, err := s.kubelet.GetAllPods()

	// when kubelet recovers from crash, podList may be empty.
	if err != nil || len(podList.Items) == 0 {
		klog.Warningf("get pods from kubelet failed, err: %v", err)
		return err
	}
	newPodMap := make(map[string]*statesinformer.PodMeta, len(podList.Items))
	// reset pod container metrics
	resetPodMetrics()
	for i := range podList.Items {
		pod := &podList.Items[i]
		podMeta := &statesinformer.PodMeta{
			Pod:       pod, // no need to deep-copy from unmarshalled
			CgroupDir: genPodCgroupParentDir(pod),
		}
		// 同步pod 元数据到newPodMap
		newPodMap[string(pod.UID)] = podMeta
		// record pod container metrics
		// 记录pod 中容器的指标
		recordPodResourceMetrics(podMeta)
	}
	s.podRWMutex.Lock()
	s.podMap = newPodMap
	s.podRWMutex.Unlock()

	s.podHasSynced.Store(true)
	s.podUpdatedTime = time.Now()
	klog.V(4).Infof("get pods success, len %d, time %s", len(s.podMap), s.podUpdatedTime.String())
	s.callbackRunner.SendCallback(statesinformer.RegisterTypeAllPods)
	return nil
}

```

recordPodResourceMetrics 负责同步的request和limit的batch-cpu、batch-memory、mid-cpu、mid-memory 到prometheus 指标中

```
func recordPodResourceMetrics(podMeta *statesinformer.PodMeta) {
	if podMeta == nil || podMeta.Pod == nil {
		klog.V(5).Infof("failed to record pod resources metric, pod is invalid: %v", podMeta)
		return
	}
	pod := podMeta.Pod

	// record (regular) container metrics
	containerStatusMap := map[string]*corev1.ContainerStatus{}
	for i := range pod.Status.ContainerStatuses {
		containerStatus := &pod.Status.ContainerStatuses[i]
		containerStatusMap[containerStatus.Name] = containerStatus
	}
	for i := range pod.Spec.Containers {
		c := &pod.Spec.Containers[i]
		containerStatus, ok := containerStatusMap[c.Name]
		if !ok {
			klog.V(6).Infof("skip record container resources metric, container %s/%s/%s status not exist",
				pod.Namespace, pod.Name, c.Name)
			continue
		}
		recordContainerResourceMetrics(c, containerStatus, pod)
	}

	klog.V(6).Infof("record pod prometheus metrics successfully, pod %s/%s", pod.Namespace, pod.Name)
}
```

#### (6) nodeMetricInformer

这快代码位于

```
pkg/koordlet/statesinformer/impl/states_nodemetric.go
```

监听nodeMetric变化

```
r.nodeMetricInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			nodeMetric, ok := obj.(*slov1alpha1.NodeMetric)
			if ok {
				r.updateMetricSpec(nodeMetric)
			} else {
				klog.Errorf("node metric informer add func parse nodeMetric failed")
			}
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldNodeMetric, oldOK := oldObj.(*slov1alpha1.NodeMetric)
			newNodeMetric, newOK := newObj.(*slov1alpha1.NodeMetric)
			if !oldOK || !newOK {
				klog.Errorf("unable to convert object to *slov1alpha1.NodeMetric, old %T, new %T", oldObj, newObj)
				return
			}

			if newNodeMetric.Generation == oldNodeMetric.Generation || reflect.DeepEqual(oldNodeMetric.Spec, newNodeMetric.Spec) {
				klog.V(5).Infof("find nodeMetric spec %s has not changed.", newNodeMetric.Name)
				return
			}
			klog.V(2).Infof("update node metric spec %v", newNodeMetric.Spec)
			r.updateMetricSpec(newNodeMetric)
		},
	})
```

更新到内存中:

```

func (r *nodeMetricInformer) updateMetricSpec(newNodeMetric *slov1alpha1.NodeMetric) {
	r.rwMutex.Lock()
	defer r.rwMutex.Unlock()
	if newNodeMetric == nil {
		klog.Error("failed to merge with nil nodeMetric, new is nil")
		return
	}
	r.nodeMetric = newNodeMetric.DeepCopy()
	data, _ := json.Marshal(newNodeMetric.Spec)
	r.nodeMetric.Spec = *defaultNodeMetricSpec.DeepCopy()
	_ = json.Unmarshal(data, &r.nodeMetric.Spec)
}
```

nodeMetricInformer 还会定时同步metric到 prodPredictor 和 statusUpdater。
nodeMetricInformer的collectMetric是一个十分重要的函数

```
// 初始化峰值预测器
prodPredictor := r.predictorFactory.New(prediction.ProdReclaimablePredictor)

	// 遍历pod指标
	for _, podMeta := range podsMeta {
		podMetric, err := r.collectPodMetric(podMeta, queryParam)
		if err != nil {
			klog.Warningf("query pod metric failed, pod %s, err: %v", podMeta.Key(), err)
			continue
		}
		// predict pods which have valid metrics; ignore prediction failures
		// 将pod 信息加入峰值预测器中
		err = prodPredictor.AddPod(podMeta.Pod)
		if err != nil {
			klog.V(4).Infof("predictor add pod aborted, pod %s, err: %v", podMeta.Key(), err)
		}

		r.fillExtensionMap(podMetric, podMeta.Pod)
		// 填充gpu 信息到pod中
		if len(gpus) > 0 {
			r.fillGPUMetrics(queryParam, podMetric, string(podMeta.Pod.UID), gpus)
		}
		podsMetricInfo = append(podsMetricInfo, podMetric)
	}
```

同步到statusUpdater，nodeMetricInformer->sync

```
//收集指标
nodeMetricInfo, podMetricInfo, hostAppMetricInfo, prodReclaimableMetric := r.collectMetric()
	if nodeMetricInfo == nil {
		klog.Warningf("node metric is not ready, skip this round.")
		return
	}

// 初始化 NodeMetricStatus
newStatus := &slov1alpha1.NodeMetricStatus{
	UpdateTime:            &metav1.Time{Time: time.Now()},
	NodeMetric:            nodeMetricInfo,
	PodsMetric:            podMetricInfo,
	HostApplicationMetric: hostAppMetricInfo,
	ProdReclaimableMetric: prodReclaimableMetric,
}

retErr := retry.RetryOnConflict(retry.DefaultBackoff, func() error {
	nodeMetric, err := r.nodeMetricLister.Get(r.nodeName)
	if errors.IsNotFound(err) {
		klog.Warningf("nodeMetric %v not found, skip", r.nodeName)
		return nil
	} else if err != nil {
		klog.Warningf("failed to get %s nodeMetric: %v", r.nodeName, err)
		return err
	}
	// 更新status
	err = r.statusUpdater.updateStatus(nodeMetric, newStatus)
	return err
})

```

### CGROUP 驱动

安装cgroup驱动

```
cgroupDriver := system.GetCgroupDriver()
	system.SetupCgroupPathFormatter(cgroupDriver)
```

判断cgroup 驱动是systemd 还是cgroupfs,其实就是判断存在kubepods 或者 kubepods.slice

```
func GetCgroupDriverFromCgroupName() CgroupDriverType {
	isSystemd := FileExists(filepath.Join(GetRootCgroupSubfsDir(CgroupCPUDir), KubeRootNameSystemd))
	if isSystemd {
		return Systemd
	}

	isCgroupfs := FileExists(filepath.Join(GetRootCgroupSubfsDir(CgroupCPUDir), KubeRootNameCgroupfs))
	if isCgroupfs {
		return Cgroupfs
	}

	return ""
}

```

### MetricAdvisor

核心代码位于:

```
pkg/koordlet/metricsadvisor/metrics_advisor.go
```

Metric Advisor 提供节点、Pod 和容器的资源使用和性能特征的基本信息。 它是一个独立的模块，定期收集、处理和导出资源画像。它还检测运行容器的干扰，例如 CPU 调度、内存分配延迟和压力阻塞信息（Pressure Stall Information, PSI）。 该信息将广泛用于资源超卖和 QoS 保障插件。

metricAdvisor 目前支持的采集模块

```
// 收集gpu 数据
devicePlugins = map[string]framework.DeviceFactory{
	gpu.DeviceCollectorName: gpu.New,
}

collectorPlugins = map[string]framework.CollectorFactory{
	// 收集节点资源
	noderesource.CollectorName:       noderesource.New,
	beresource.CollectorName:         beresource.New,
	nodeinfo.CollectorName:           nodeinfo.New,
	nodestorageinfo.CollectorName:    nodestorageinfo.New,
	podresource.CollectorName:        podresource.New,
	podthrottled.CollectorName:       podthrottled.New,
	performance.CollectorName:        performance.New,
	sysresource.CollectorName:        sysresource.New,
	coldmemoryresource.CollectorName: coldmemoryresource.New,
	pagecache.CollectorName:          pagecache.New,
	hostapplication.CollectorName:    hostapplication.New,
}

```

调用metricAdvisor->run后运行所有模块采集

```
for name, dc := range m.context.DeviceCollectors {
	klog.V(4).Infof("ready to start device collector %v", name)
	if !dc.Enabled() {
		klog.V(4).Infof("device collector %v is not enabled, skip running", name)
		continue
	}
	go dc.Run(stopCh)
	klog.V(4).Infof("device collector %v start", name)
}

for name, collector := range m.context.Collectors {
	klog.V(4).Infof("ready to start collector %v", name)
	if !collector.Enabled() {
		klog.V(4).Infof("collector %v is not enabled, skip running", name)
		continue
	}
	go collector.Run(stopCh)
	klog.V(4).Infof("collector %v start", name)
}
```

### evictVersion

eviction，即驱赶的意思，意思是当节点出现异常时，kubernetes将有相应的机制驱赶该节点上的Pod。eviction在openstack的nova组件中也存在。

目前kubernetes中存在两种eviction机制，分别由kube-controller-manager和kubelet实现。

koordinator 使用 FindSupportedEvictVersion发现驱逐器版本。

### qosManager