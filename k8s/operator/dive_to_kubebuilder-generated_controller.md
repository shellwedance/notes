# How custom controllers created by Kubebuilder work


Kubebuilder를 통해 생성된 코드에서 어떻게 controller가 작동하고 이벤트 발생 시 reconcile을 수행하는지 code level에서 분석합니다.

예제로는 Kubebuilder를 통해 생성한 [HyperSDS-Operator](https://www.github.com/tmax-cloud/hypersds-operator) 프로젝트를 사용합니다.


### 1. Kubebuilder가 자동으로 만들어주는 main 함수<a name="sec1"></a>

아래는 Kubebuilder로 생성한 project인 HyperSDS-Operator의 main 함수
- Manager를 생성하여 CephCluster controller를 등록하고 manager를 start함

```go
// hypersds-operator/main.go

mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
	Scheme:             scheme,
	MetricsBindAddress: metricsAddr,
	Port:               metricsPort,
	LeaderElection:     enableLeaderElection,
	LeaderElectionID:   "35b060ce.tmax.io",
})

...

if err = (&controllers.CephClusterReconciler{
	Client: mgr.GetClient(),
	Log:    ctrl.Log.WithName("controllers").WithName("CephCluster"),
	Scheme: mgr.GetScheme(),
}).SetupWithManager(mgr); err != nil {
	setupLog.Error(err, "unable to create controller", "controller", "CephCluster")
	os.Exit(1)
}

...

setupLog.Info("starting manager")
if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
	setupLog.Error(err, "problem running manager")
	os.Exit(1)
}
```

NewManager 함수는 아래와 같이 alias 됨

```go
// controller-runtime/alias.go

...

// NewManager returns a new Manager for creating Controllers.
NewManager = manager.New
```

Manager의 New 함수는 아래와 같음
- Cache, client 등 manager의 구성 요소들을 초기화하며 새로운 manager 생성

```go
// controller-runtime/pkg/manager/manager.go

// New returns a new Manager for creating Controllers.
func New(config *rest.Config, options Options) (Manager, error) {
    // Set default values for options fields
    options = setOptionsDefaults(options)

    cluster, err := cluster.New(config, func(clusterOptions *cluster.Options) {
        clusterOptions.Scheme = options.Scheme
        clusterOptions.MapperProvider = options.MapperProvider
        clusterOptions.Logger = options.Logger
        clusterOptions.SyncPeriod = options.SyncPeriod
        clusterOptions.Namespace = options.Namespace
        clusterOptions.NewCache = options.NewCache
        clusterOptions.NewClient = options.NewClient
        clusterOptions.ClientDisableCacheFor = options.ClientDisableCacheFor
        clusterOptions.DryRunClient = options.DryRunClient
        clusterOptions.EventBroadcaster = options.EventBroadcaster
    })
    
    ...
```


### 2. Kubebuilder가 자동으로 생성해주는 controller 코드<a name="sec2"></a>

```go
// hypersds-operator/controller/cephcluster_controller.go

import (
	ctrl "sigs.k8s.io/controller-runtime"
)

...

func (r *CephClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&hypersdsv1alpha1.CephCluster{}).
		Watches(&source.Kind{Type: &corev1.ConfigMap{}},
			&handler.EnqueueRequestForOwner{IsController: true, OwnerType: &hypersdsv1alpha1.CephCluster{}}).
		Watches(&source.Kind{Type: &corev1.Pod{}},
			&handler.EnqueueRequestForOwner{IsController: true, OwnerType: &hypersdsv1alpha1.CephCluster{}}).
		...
		Watches(&source.Kind{Type: &v1.RoleBinding{}},
			&handler.EnqueueRequestForOwner{IsController: true, OwnerType: &hypersdsv1alpha1.CephCluster{}}).
		Complete(r)
}
```


NewControllerManagedBy 함수는 아래와 같이 alias 됨

```go
// controller-runtime/alias.go

// NewControllerManagedBy returns a new controller builder that will be started by the provided Manager
NewControllerManagedBy = builder.ControllerManagedBy
```


### 3. Builder 클래스

Builder는 controller-runtime 패키지들을 wrapping하여 builder pattern으로 controller를 생성해주는 패키지

<a name="builder-class"></a>
```go
// controller-runtime/pkg/builder/controller.go

// Builder builds a Controller.
type Builder struct {
    forInput         ForInput
    ownsInput        []OwnsInput
    watchesInput     []WatchesInput
    mgr              manager.Manager
    globalPredicates []predicate.Predicate
    ctrl             controller.Controller
    ctrlOptions      controller.Options
    name             string
}
```

[2번](#sec2)의 ControllerManagedBy 함수는 manager의 관리를 받는 builder 객체 주소를 return

```go
// controller-runtime/pkg/builder/controller.go

func ControllerManagedBy(m manager.Manager) *Builder {
	return &Builder{mgr: m}
}
```


즉, kubebuilder가 생성한 위 [2번](#sec2)의 SetupWithManager의 실제 코드는 아래와 같음
- Builder의 For, Owns, Watches, Completes를 수행
- `EnqueueRequestForOwner`는 source object에 event 발생 시 Owner에게 request를 enqueue함
  - ReplicaSet이 Pod event 감지하면 reconcile 하는데, 이때 `source.Kind`는 Pod이고, `OwnerType`은 ReplicaSet임

```go
func (r *CephClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return Builder(mgr).ControllerManagedBy(mgr).
		For(&hypersdsv1alpha1.CephCluster{}).
		Watches(&source.Kind{Type: &corev1.ConfigMap{}},
				&handler.EnqueueRequestForOwner{IsController: true, OwnerType: &hypersdsv1alpha1.CephCluster{}}).
		Watches(&source.Kind{Type: &corev1.Pod{}},
				&handler.EnqueueRequestForOwner{IsController: true, OwnerType: &hypersdsv1alpha1.CephCluster{}}).
		...
		Complete(r)
```


아래는 Builder의 Watches 함수 예시
- Watch 할 input을 watchesInput list에 append ([Builder 클래스 참고](#builder-class))
- 나중에 호출하는 Completes 함수를 통해 이 watchesInput list의 resource들에 Watch 호출

```go
// controller-runtime/pkg/builder/controller.go

// Watches exposes the lower-level ControllerManagedBy Watches functions through the builder.  Consider using
// Owns or For instead of Watches directly.
// Specified predicates are registered only for given source.
func (blder *Builder) Watches(src source.Source, eventhandler handler.EventHandler, opts ...WatchesOption) *Builder {
    input := WatchesInput{src: src, eventhandler: eventhandler}
    for _, opt := range opts {
        opt.ApplyToWatches(&input)
    }

    blder.watchesInput = append(blder.watchesInput, input)
    return blder
}
```

- For, Owns 모두 controller가 watch하는 대상
  - For는 Reconcile 대상
  - Owns는 controller가 생성한 object 대상
  - Watches는 아무 조건 없는 watch 함수로, For와 Owns 모두 Watches로 대체할 수 있음


### 4. Builder의 Complete 및 Build 함수

For, Owns, Watches 함수 등을 호출한 후 최종적으로 Completes를 호출하여 위에서 정의된 controller를 빌드함

```go
// controller-runtime/pkg/builder/controller.go

// Complete builds the Application Controller.
func (blder *Builder) Complete(r reconcile.Reconciler) error {
    _, err := blder.Build(r)
    return err
}
```


Builder의 Build 함수는 크게 두 함수를 통해 실제 controller를 빌드함
- doController 함수를 통해 실제 controller를 생성
- doWatch 함수를 통해 관리할 resource에 대한 Watch를 수행


```go
// controller-runtime/pkg/builder/controller.go

// Build builds the Application Controller and returns the Controller it created.
func (blder *Builder) Build(r reconcile.Reconciler) (controller.Controller, error) {
	...

    // Set the ControllerManagedBy
    if err := blder.doController(r); err != nil {
        return nil, err
    }

    // Set the Watch
    if err := blder.doWatch(); err != nil {
        return nil, err
    }

    return blder.ctrl, nil
}
```


### 5. Builder의 doController 함수<a name="sec5"></a>

Reconciler, concurrency, cache timeout 등이 별도로 설정되지 않다면 default로 설정하고 controller를 생성


```go
// controller-runtime/pkg/builder/controller.go

func (blder *Builder) doController(r reconcile.Reconciler) error {
    // Setup concurrency, cache timeout, log
    ...
	
    // Build the controller and return.
    blder.ctrl, err = newController(blder.getControllerName(gvk), blder.mgr, ctrlOptions)
    return err
}
```


위의 newController는 아래와 같이 alias돼있음


```go
// controller-runtime/alias.go

import(
	...
	"sigs.k8s.io/controller-runtime/pkg/controller"
)

...

var newController = controller.New
```

### 6. Controller의 New 함수<a name="sec6"></a>

실제 controller를 생성하고 해당 controller를 controller manager에 등록
- NewUnmanaged 함수를 통해 실제 controller를 생성
- Manager Add 함수를 통해 입력받은 manager에 해당 controller를 등록

```go
// controller-runtime/pkg/controller/controller.go

// New returns a new Controller registered with the Manager.  The Manager will ensure that shared Caches have
// been synced before the Controller is Started.
func New(name string, mgr manager.Manager, options Options) (Controller, error) {
    c, err := NewUnmanaged(name, mgr, options)
    if err != nil {
        return nil, err
    }

    // Add the controller as a Manager components
    return c, mgr.Add(c)
}
```


### 7. Controller의 NewUnmanaged 함수

실제 controller를 생성
- Controller의 workqueue가 여기서 생성됨
- 해당 controller의 최대 reconcile concurrency가 설정됨

```go
// controller-runtime/pkg/controller/controller.go

// NewUnmanaged returns a new controller without adding it to the manager. The
// caller is responsible for starting the returned controller.
func NewUnmanaged(name string, mgr manager.Manager, options Options) (Controller, error) {
    ...

    // Create controller with dependencies set
    return &controller.Controller{
        Do: options.Reconciler,
        MakeQueue: func() workqueue.RateLimitingInterface {
            return workqueue.NewNamedRateLimitingQueue(options.RateLimiter, name)
        },
        MaxConcurrentReconciles: options.MaxConcurrentReconciles,
        CacheSyncTimeout:        options.CacheSyncTimeout,
        SetFields:               mgr.SetFields,
        Name:                    name,
        Log:                     options.Log.WithName("controller").WithName(name),
    }, nil
}
```


### 8. Manager의 Add 함수

[6번](#sec6)에서 NewUnmanaged 함수 호출 후에 호출하는 manager의 Add 함수는 아래와 같이 정의됨

```go
// controller-runtime/pkg/manager/manager.go

// Manager initializes shared dependencies such as Caches and Clients, and provides them to Runnables.
// A Manager is required to create Controllers.
type Manager interface {
	...
	
    // Add will set requested dependencies on the component, and cause the component to be
    // started when Start is called.  Add will inject any dependencies for which the argument
    // implements the inject interface - e.g. inject.Client.
    // Depending on if a Runnable implements LeaderElectionRunnable interface, a Runnable can be run in either
    // non-leaderelection mode (always running) or leader election mode (managed by leader election if enabled).
    Add(Runnable) error
	
	...
}
```


Runnable은 Start라는 메소드를 가지는 Interface
- Controller가 이를 구현하고 있으므로 Add 함수를 통해 manager에 controller를 add할 수 있음


```go
// controller-runtime/pkg/manager/manager.go

// Runnable allows a component to be started.
// It's very important that Start blocks until
// it's done running.
type Runnable interface {
    // Start starts running the component.  The component will stop running
    // when the context is closed. Start blocks until the context is closed or
    // an error occurs.
    Start(context.Context) error
}
```


Manager의 Add 함수에서는 controller를 start list에 추가해줌 
- 이미 manager가 start된 경우 controller를 바로 start (cm.startRunnable -> r.Start)

```go
// controller-runtime/pkg/manager/internal.go

// Add sets dependencies on i, and adds it to the list of Runnables to start.
func (cm *controllerManager) Add(r Runnable) error {
    ...

    var shouldStart bool

    // Add the runnable to the leader election or the non-leaderelection list
    if leRunnable, ok := r.(LeaderElectionRunnable); ok && !leRunnable.NeedLeaderElection() {
        shouldStart = cm.started
        cm.nonLeaderElectionRunnables = append(cm.nonLeaderElectionRunnables, r)
    } else if hasCache, ok := r.(hasCache); ok {
        cm.caches = append(cm.caches, hasCache)
    } else {
        shouldStart = cm.startedLeader
        cm.leaderElectionRunnables = append(cm.leaderElectionRunnables, r)
    }

    if shouldStart {
        // If already started, start the controller
        cm.startRunnable(r)
    }

    return nil
}

func (cm *controllerManager) startRunnable(r Runnable) {
    cm.waitForRunnable.Add(1)
    go func() {
        defer cm.waitForRunnable.Done()
        if err := r.Start(cm.internalCtx); err != nil {
            cm.errChan <- err
        }
    }()
}
```

### 9. Builder의 doWatch 함수

[5번](#sec5)의 doController가 이렇게 다 끝나고 나면 doWatch를 수행
- doWatch를 통해 For, Owns, Watches를 호출한 모든 대상 object들에 Watch 

```
// controller-runtime/pkg/builder/controller.go

func (blder *Builder) doWatch() error {
    // Reconcile type
    typeForSrc, err := blder.project(blder.forInput.object, blder.forInput.objectProjection)
    if err != nil {
        return err
    }
    src := &source.Kind{Type: typeForSrc}
    hdler := &handler.EnqueueRequestForObject{}
    allPredicates := append(blder.globalPredicates, blder.forInput.predicates...)
    if err := blder.ctrl.Watch(src, hdler, allPredicates...); err != nil {
        return err
    }

    // Watches the managed types
    for _, own := range blder.ownsInput {
        typeForSrc, err := blder.project(own.object, own.objectProjection)
        if err != nil {
            return err
        }
        src := &source.Kind{Type: typeForSrc}
        hdler := &handler.EnqueueRequestForOwner{
            OwnerType:    blder.forInput.object,
            IsController: true,
        }
        allPredicates := append([]predicate.Predicate(nil), blder.globalPredicates...)
        allPredicates = append(allPredicates, own.predicates...)
        if err := blder.ctrl.Watch(src, hdler, allPredicates...); err != nil {
            return err
        }
    }

    // Do the watch requests
    for _, w := range blder.watchesInput {
        allPredicates := append([]predicate.Predicate(nil), blder.globalPredicates...)
        allPredicates = append(allPredicates, w.predicates...)

        // If the source of this watch is of type *source.Kind, project it.
        if srckind, ok := w.src.(*source.Kind); ok {
            typeForSrc, err := blder.project(srckind.Type, w.objectProjection)
            if err != nil {
                return err
            }
            srckind.Type = typeForSrc
        }

        if err := blder.ctrl.Watch(w.src, w.eventhandler, allPredicates...); err != nil {
            return err
        }
    }
    return nil
}
```

controller.Controller의 Watch 함수는 source.Source 타입인 src에 대한 cache를 생성하고 src의 Start 함수를 호출
- [2번](#sec2)에서는 `&source.Kind{Type: &corev1.ConfigMap{}}` 등을 Watches 호출했었음 (src가 source.Kind)

```go
// controller-runtime/pkg/internal/controller/controller.go

// Watch implements controller.Controller
func (c *Controller) Watch(src source.Source, evthdler handler.EventHandler, prct ...predicate.Predicate) error {
    ...
    
    return src.Start(c.ctx, evthdler, c.Queue, prct...)
}
```

source.Kind의 Start 함수는 manager cache의 informer에 controller의 workqueue와 handler([2번](#sec2)에서 설정)를 등록함

```go
// controller-runtime/pkg/source/source.go

// Start is internal and should be called only by the Controller to register an EventHandler with the Informer
// to enqueue reconcile.Requests.
func (ks *Kind) Start(ctx context.Context, handler handler.EventHandler, queue workqueue.RateLimitingInterface,
    prct ...predicate.Predicate) error {
    
    ...

    // cache.GetInformer will block until its context is cancelled if the cache was already started and it can not
    // sync that informer (most commonly due to RBAC issues).
    ctx, ks.startCancel = context.WithCancel(ctx)
    ks.started = make(chan error)
    go func() {
        // Lookup the Informer from the Cache and add an EventHandler which populates the Queue
        i, err := ks.cache.GetInformer(ctx, ks.Type)
        if err != nil {
            kindMatchErr := &meta.NoKindMatchError{}
            if errors.As(err, &kindMatchErr) {
                log.Error(err, "if kind is a CRD, it should be installed before calling Start",
                    "kind", kindMatchErr.GroupKind)
            }
            ks.started <- err
            return
        }
        i.AddEventHandler(internal.EventHandler{Queue: queue, EventHandler: handler, Predicates: prct})
        if !ks.cache.WaitForCacheSync(ctx) {
            // Would be great to return something more informative here
            ks.started <- errors.New("cache did not sync")
        }
        close(ks.started)
    }()

    return nil
}
```


### 10. Manager의 Start 함수

위와 같은 과정으로 controller 생성 및 Watch 설정까지 마친 후 [1번](#sec1)과 같이 manager Start를 하면 add한 controller를 아래와 같이 start시킴

```go
// controller-runtime/pkg/manager/internal.go

func (cm *controllerManager) Start(ctx context.Context) (err error) {
    if err := cm.Add(cm.cluster); err != nil {
        return fmt.Errorf("failed to add cluster to runnables: %w", err)
    }
    cm.internalCtx, cm.internalCancel = context.WithCancel(ctx)

    // This chan indicates that stop is complete, in other words all runnables have returned or timeout on stop request
    stopComplete := make(chan struct{})
    defer close(stopComplete)
	
    ...

    // initialize this here so that we reset the signal channel state on every start
    // Everything that might write into this channel must be started in a new goroutine,
    // because otherwise we might block this routine trying to write into the full channel
    // and will not be able to enter the deferred cm.engageStopProcedure() which drains
    // it.
    cm.errChan = make(chan error)

    ...
	
    go cm.startNonLeaderElectionRunnables()

    go func() {
        if cm.resourceLock != nil {
            err := cm.startLeaderElection()
            if err != nil {
                cm.errChan <- err
            }
        } else {
            // Treat not having leader election enabled the same as being elected.
            cm.startLeaderElectionRunnables()
            close(cm.elected)
        }
    }()

    select {
    case <-ctx.Done():
        // We are done
        return nil
    case err := <-cm.errChan:
        // Error starting or running a runnable
        return err
    }
}
```

Manager의 Start 함수를 타고 가면 controller.Controller의 Start 함수를 호출함

### 11. Controller의 Start 함수

Controller의 Start 함수에서는 workqueue에 request가 들어오기를 기다리다가 들어오면 처리함
- MaxConcurrentReconciles 만큼 processNextWorkItem 함수를 수행하는 worker thread를 async로 띄움

```go
// controller-runtime/pkg/internal/controller/controller.go

func (c *Controller) Start(ctx context.Context) error {
    ...
    err := func() error {
	// Launch workers to process resources
	c.Log.Info("Starting workers", "worker count", c.MaxConcurrentReconciles)
	wg.Add(c.MaxConcurrentReconciles)
	for i := 0; i < c.MaxConcurrentReconciles; i++ {
	    go func() {
		defer wg.Done()
		// Run a worker thread that just dequeues items, processes them, and marks them done.
		// It enforces that the reconcileHandler is never invoked concurrently with the same object.
		for c.processNextWorkItem(ctx) {
		}
	    }()
	}

	c.Started = true
	return nil
    }()
	
    ...
	
    return nil
}
```

processNextWorkItem 함수에서 reconcileHandler를 호출
- Kubebuilder에서 생성한 Reconcile 함수에 object를 넘겨줌

```go
// controller-runtime/pkg/internal/controller/controller.go

// processNextWorkItem will read a single work item off the workqueue and
// attempt to process it, by calling the reconcileHandler.
func (c *Controller) processNextWorkItem(ctx context.Context) bool {
    obj, shutdown := c.Queue.Get()
    if shutdown {
        // Stop working
        return false
    }

    ...

    c.reconcileHandler(ctx, obj)
    return true
}
```

## Reference

https://ssup2.github.io/programming/Kubernetes_Kubebuilder/

https://jishuin.proginn.com/p/763bfbd3012b

https://juejin.cn/post/6858058420453539853
