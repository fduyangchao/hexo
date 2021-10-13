---
title: Volume Manager源码分析
date: 2021/10/12 22:10:01
categories:
  - 专业
  - 开发
  - Kubernetes
tags:
  - Kubernetes
  - 云原生
  - golang
---
工作中需要调研k8s存储插件从in-tree模式平滑升级到CSI的可行性，因此对Kubelet中的Volume Manager模块进行源码分析，期望在全面理解代码逻辑和设计思路的前提下，避免采坑。按照一贯的思路和习惯，从接口、数据结构、函数/方法三个方面入手。

<!-- more -->

# 接口

- 运行在kubelet 里让存储Ready的部件，主要是mount/unmount（attach/detach可选）
- pod调度到这个node上后才会有卷的相应操作，所以它的触发端是kubelet（严格讲是kubelet里的pod manager），根据Pod Manager里pod spec里申明的存储来触发卷的挂载操作
- Kubelet监听到调度到该节点上的pod声明，把pod缓存到Pod Manager中，VolumeManager通过Pod Manager获取PV/PVC的状态，并进行分析出具体的attach/detach、mount/umount操作然后调用plugin进行相应的业务处理

```go
// VolumeManager runs a set of asynchronous loops that figure out which volumes
// need to be attached/mounted/unmounted/detached based on the pods scheduled on
// this node and makes it so.
type VolumeManager interface {
 
	Run(sourcesReady config.SourcesReady, stopCh <-chan struct{})
 
	WaitForAttachAndMount(pod *v1.Pod) error
 
	GetMountedVolumesForPod(podName types.UniquePodName) container.VolumeMap
 
	GetExtraSupplementalGroupsForPod(pod *v1.Pod) []int64
 
	GetVolumesInUse() []v1.UniqueVolumeName
 
	ReconcilerStatesHasBeenSynced() bool
 
	VolumeIsAttached(volumeName v1.UniqueVolumeName) bool
 
	MarkVolumesAsReportedInUse(volumesReportedAsInUse []v1.UniqueVolumeName)

```



# 数据结构

```go
// volumeManager implements the VolumeManager interface
type volumeManager struct {
	
	kubeClient clientset.Interface			// dswPopulator通过kubeClient获取pvc和pv对象
	// volumePluginMgr用于管理存储插件
	volumePluginMgr *volume.VolumePluginMgr	// volumePluginMgr用于管理存储插件
	
	desiredStateOfWorld cache.DesiredStateOfWorld	// 期望状态
	
	actualStateOfWorld cache.ActualStateOfWorld		// 当前状态
	operationExecutor operationexecutor.OperationExecutor	// 开始异步attach, detach, mount, unmount等操作

	reconciler reconciler.Reconciler	// Rconciler运行异步周期性的调谐逻辑，调谐dsw和asw的状态，出发attach, detach, mount, umount等操作

	desiredStateOfWorldPopulator populator.DesiredStateOfWorldPopulator		// 运行一个异步的周期循环，通过kubelet pod manager计算dsw
}

```

volumeManager结构体实现了VolumeManager接口，主要有两个数据结构需要注意：

- desiredStateOfWorld：预期状态，表征哪些volume需要被attach，哪些pods引用这个volume。dsw由dswPopulator通过kubelet pod manager计算并更新。
- actualStateOfWorld：当前状态，表征哪些volume已经被attach到node，这些volume被mount到哪些pod。asw由reconcile函数的attach, dtach, mount, umount的成功执行操作更新

**`desiredStateOfWorld`**

```go
type desiredStateOfWorld struct {
	volumesToMount map[v1.UniqueVolumeName]volumeToMount
	volumePluginMgr *volume.VolumePluginMgr
	sync.RWMutex
}

// The volume object represents a volume that should be attached to this node,
// and mounted to podsToMount.
type volumeToMount struct {
	// volumeName contains the unique identifier for this volume.
	volumeName v1.UniqueVolumeName

	// podsToMount is a map containing the set of pods that reference this
	// volume and should mount it once it is attached. The key in the map is
	// the name of the pod and the value is a pod object containing more
	// information about the pod.
	podsToMount map[types.UniquePodName]podToMount

	// pluginIsAttachable indicates that the plugin for this volume implements
	// the volume.Attacher interface
	pluginIsAttachable bool

	// volumeGidValue contains the value of the GID annotation, if present.
	volumeGidValue string

	// reportedInUse indicates that the volume was successfully added to the
	// VolumesInUse field in the node's status.
	reportedInUse bool
}

// The pod object represents a pod that references the underlying volume and
// should mount it once it is attached.
type podToMount struct {
	// podName contains the name of this pod.
	podName types.UniquePodName

	// Pod to mount the volume to. Used to create NewMounter.
	pod *v1.Pod

	// volume spec containing the specification for this volume. Used to
	// generate the volume plugin object, and passed to plugin methods.
	// For non-PVC volumes this is the same as defined in the pod object. For
	// PVC volumes it is from the dereferenced PV object.
	spec *volume.Spec

	// outerVolumeSpecName is the volume.Spec.Name() of the volume as referenced
	// directly in the pod. If the volume was referenced through a persistent
	// volume claim, this contains the volume.Spec.Name() of the persistent
	// volume claim
	outerVolumeSpecName string
}

```

**`actualStateOfWorld`**

```go
type actualStateOfWorld struct {
	nodeName types.NodeName
	attachedVolumes map[v1.UniqueVolumeName]attachedVolume
	volumePluginMgr *volume.VolumePluginMgr
	sync.RWMutex
}
```



# 功能实现

## volumeManager初始化

volumeManger结构体初始化

```go
// setup volumeManager
	klet.volumeManager = volumemanager.NewVolumeManager(
		kubeCfg.EnableControllerAttachDetach,
		nodeName,
		klet.podManager,
		klet.statusManager,
		klet.kubeClient,
		klet.volumePluginMgr,
		klet.containerRuntime,
		kubeDeps.Mounter,
		klet.getPodsDir(),
		kubeDeps.Recorder,
		experimentalCheckNodeCapabilitiesBeforeMount,
		keepTerminatedPodVolumes)
```



## 启动volumeManger.Run()

- goroutine启动dswPopulator.Run()：从apiserver同步pod信息，更新dsw
- goroutine启动reconcile.Run()：调谐预期状态和当前状态，将当前状态手链到预期状态

### desiredStateOfWorldPopulator.Run()

dswp.Run()启动一个loop定时调用dswp.populatorLoop()，用于更新dsw

```

func (dswp *desiredStateOfWorldPopulator) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
	// Wait for the completion of a loop that started after sources are all ready, then set hasAddedPods accordingly
	glog.Infof("Desired state populator starts to run")
	wait.PollUntil(dswp.loopSleepDuration, func() (bool, error) {
		done := sourcesReady.AllReady()
		dswp.populatorLoop()
		return done, nil
	}, stopCh)
	dswp.hasAddedPodsLock.Lock()
	dswp.hasAddedPods = true
	dswp.hasAddedPodsLock.Unlock()
	wait.Until(dswp.populatorLoop, dswp.loopSleepDuration, stopCh)
}
```

#### dswp.populatorLoop()

dswp.populatorLoop()先调用dswp.findAndAddNewPods()，获取所有pods并更新到dsw

```go
func (dswp *desiredStateOfWorldPopulator) populatorLoop() {
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
```

##### dsw.findAndAddNewPods()

-   调用podManager获取所有的pods
-   调用processPodVolumes去更新desiredStateOfWorld

```go
// Iterate through all pods and add to desired state of world if they don't
// exist but should
func (dswp *desiredStateOfWorldPopulator) findAndAddNewPods() {
	// Map unique pod name to outer volume name to MountedVolume.
	mountedVolumesForPod := make(map[volumetypes.UniquePodName]map[string]cache.MountedVolume)
	if utilfeature.DefaultFeatureGate.Enabled(features.ExpandInUsePersistentVolumes) {
		for _, mountedVolume := range dswp.actualStateOfWorld.GetMountedVolumes() {
			mountedVolumes, exist := mountedVolumesForPod[mountedVolume.PodName]
			if !exist {
				mountedVolumes = make(map[string]cache.MountedVolume)
				mountedVolumesForPod[mountedVolume.PodName] = mountedVolumes
			}
			mountedVolumes[mountedVolume.OuterVolumeSpecName] = mountedVolume
		}
	}
 
	processedVolumesForFSResize := sets.NewString()
	for _, pod := range dswp.podManager.GetPods() {
		if dswp.isPodTerminated(pod) {
			// Do not (re)add volumes for terminated pods
			continue
		}
		dswp.processPodVolumes(pod, mountedVolumesForPod, processedVolumesForFSResize)
	}
}
```

###### dswp.processPodVolumes()

- 如果volume已经在processedPod map中，无需处理:

  ```go
  uniquePodName := util.GetUniquePodName(pod)
  	if dswp.podPreviouslyProcessed(uniquePodName) {
  		return
  }
  ```

- 对pod 下所有容器下的volumeDevice和volumeMounts加入processedPods map中:

  ```go
  volumeDevices    <[]Object>
    volumeDevices is the list of block devices to be used by the container.
    This is an alpha feature and may change in the future.
  
  volumeMounts    <[]Object>
    Pod volumes to mount into the container's filesystem. Cannot be updated.
       
  func (dswp *desiredStateOfWorldPopulator) makeVolumeMap(containers []v1.Container) (map[string]bool, map[string]bool) {
  	volumeDevicesMap := make(map[string]bool)
  	volumeMountsMap := make(map[string]bool)
   
  	for _, container := range containers {
  		if container.VolumeMounts != nil {
  			for _, mount := range container.VolumeMounts {
  				volumeMountsMap[mount.Name] = true
  			}
  		}
  		// TODO: remove feature gate check after no longer needed
  		if utilfeature.DefaultFeatureGate.Enabled(features.BlockVolume) &&
  			container.VolumeDevices != nil {
  			for _, device := range container.VolumeDevices {
  				volumeDevicesMap[device.Name] = true
  			}
  		}
  	}
   
  	return volumeMountsMap, volumeDevicesMap
  }
  ```

- AddPodToVolume()

  - 调用FindPluginBySpec函数根据volume.spec找到volume plugin

    ```go
    volumePlugin, err := dsw.volumePluginMgr.FindPluginBySpec(volumeSpec)
    	if err != nil || volumePlugin == nil {
    		return "", fmt.Errorf(
    			"failed to get Plugin from volumeSpec for volume %q err=%v",
    			volumeSpec.Name(),
    			err)
    	}
    ```

  - isAttachableVolume()，检查插件是否需要attach：不是所有的插件都需要实现AttachableVolumePlugin接口

    ```go
    // The unique volume name used depends on whether the volume is attachable
    	// or not.
    	attachable := dsw.isAttachableVolume(volumeSpec)
    	if attachable {
    		// For attachable volumes, use the unique volume name as reported by
    		// the plugin.
    		volumeName, err =
    			util.GetUniqueVolumeNameFromSpec(volumePlugin, volumeSpec)
    		if err != nil {
    			return "", fmt.Errorf(
    				"failed to GetUniqueVolumeNameFromSpec for volumeSpec %q using volume plugin %q err=%v",
    				volumeSpec.Name(),
    				volumePlugin.GetPluginName(),
    				err)
    		}
    	} else {
    		// For non-attachable volumes, generate a unique name based on the pod
    		// namespace and name and the name of the volume within the pod.
    		volumeName = util.GetUniqueVolumeNameForNonAttachableVolume(podName, volumePlugin, volumeSpec)
    	}
    ```

  - 记录volume与pod之间的关系到dsw

    ```go
    if _, volumeExists := dsw.volumesToMount[volumeName]; !volumeExists {
    		dsw.volumesToMount[volumeName] = volumeToMount{
    			volumeName:              volumeName,
    			podsToMount:             make(map[types.UniquePodName]podToMount),
    			pluginIsAttachable:      attachable,
    			pluginIsDeviceMountable: deviceMountable,
    			volumeGidValue:          volumeGidValue,
    			reportedInUse:           false,
    		}
    	}
     
    	// Create new podToMount object. If it already exists, it is refreshed with
    	// updated values (this is required for volumes that require remounting on
    	// pod update, like Downward API volumes).
    	dsw.volumesToMount[volumeName].podsToMount[podName] = podToMount{
    		podName:             podName,
    		pod:                 pod,
    		volumeSpec:          volumeSpec,
    		outerVolumeSpecName: outerVolumeSpecName,
    	}
    ```

  - 对pod name标记为已处理，aws标记重新挂载

    ```go
    // some of the volume additions may have failed, should not mark this pod as fully processed
    	if allVolumesAdded {
    		dswp.markPodProcessed(uniquePodName)
    		// New pod has been synced. Re-mount all volumes that need it
    		// (e.g. DownwardAPI)
    		dswp.actualStateOfWorld.MarkRemountRequired(uniquePodName)
    	}
    ```



## reconcile.Run()

`reconcile()`的实现逻辑如下：

- 对于实际已经挂载的与预期不一样的需要unmount：UnmountVolume函数中处理，volume分为filesystem与block，在dsw不包括asw的情况需要unmount

  ```Go
  // Ensure volumes that should be unmounted are unmounted.
  	for _, mountedVolume := range rc.actualStateOfWorld.GetMountedVolumes() {
  		if !rc.desiredStateOfWorld.PodExistsInVolume(mountedVolume.PodName, mountedVolume.VolumeName) {
  			// Volume is mounted, unmount it
  			glog.V(5).Infof(mountedVolume.GenerateMsgDetailed("Starting operationExecutor.UnmountVolume", ""))
  			err := rc.operationExecutor.UnmountVolume(
  				mountedVolume.MountedVolume, rc.actualStateOfWorld, rc.kubeletPodsDir)
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

- 从dsw数据结构中获取需要mount的volumes

  ```go
  	// Ensure volumes that should be attached/mounted are attached/mounted.
  	for _, volumeToMount := range rc.desiredStateOfWorld.GetVolumesToMount() {
  		volMounted, devicePath, err := rc.actualStateOfWorld.PodExistsInVolume(volumeToMount.PodName, volumeToMount.VolumeName)
  		volumeToMount.DevicePath = devicePath
  ```

- 如果volume没有attach (err为volumeNotAttachedError)，查看actualStateOfWorld结构体中attachedVolumes没有volumeName，在则调用AttachVolume函数
- 如果volume没有mount或者error为remountRequiredError， 调用MountVolume函数
- 如果err为fsResizeRequiredError，调用ExpandVolumeFSWithoutUnmounting函数



## operationExecutor初始化

NewVolumeManager函數中初始化operationExecutor的函數爲NewOperationExecutor。operationExecuter根据不同的volume plugin生成mountvolume, attachvolume函数，供reconcile调用。

```go

		operationExecutor: operationexecutor.NewOperationExecutor(operationexecutor.NewOperationGenerator(
			kubeClient,
			volumePluginMgr,
			recorder,
			checkNodeCapabilitiesBeforeMount,
			volumepathhandler.NewBlockVolumePathHandler())),
 
 
// NewOperationExecutor returns a new instance of OperationExecutor.
func NewOperationExecutor(
	operationGenerator OperationGenerator) OperationExecutor {
 
	return &operationExecutor{
		pendingOperations: nestedpendingoperations.NewNestedPendingOperations(
			true /* exponentialBackOffOnError */),
		operationGenerator: operationGenerator,
	}
}
```

## SyncPod()

  同步 pod 时，等待 pod attach 和 mount 完成

```Go
// Volume manager will not mount volumes for terminated pods
if !kl.podIsTerminated(pod) {
	// Wait for volumes to attach/mount
	if err := kl.volumeManager.WaitForAttachAndMount(pod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedMountVolume, "Unable to mount volumes for pod %q: %v", format.Pod(pod), err)
		klog.Errorf("Unable to mount volumes for pod %q: %v; skipping pod", format.Pod(pod), err)
		return err
	}
}
```

### WaitForAttachAndMount()

 获取 pod 所有的 volume 

```
func (vm *volumeManager) WaitForAttachAndMount(pod *v1.Pod) error {
	if pod == nil {
		return nil
	}
 
	expectedVolumes := getExpectedVolumes(pod)
	if len(expectedVolumes) == 0 {
		// No volumes to verify
		return nil
	}
```

### verifyVolumesMountedFunc()

   没有被 mount 的volume 数量为0，表示成功完成挂载

```Go
// verifyVolumesMountedFunc returns a method that returns true when all expected
// volumes are mounted.
func (vm *volumeManager) verifyVolumesMountedFunc(podName types.UniquePodName, expectedVolumes []string) wait.ConditionFunc {
	return func() (done bool, err error) {
		return len(vm.getUnmountedVolumes(podName, expectedVolumes)) == 0, nil
	}
}
```



# 参考

https://blog.csdn.net/zhonglinzhang/article/details/82800287
