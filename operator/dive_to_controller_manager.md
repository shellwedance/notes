## How custom controllers created by Kubebuilder work

#### 1. Kubebuilder가 자동으로 만들어주는 main 함수<a name="sec1"></a>

아래는 Kubebuilder로 생성한 project인 HyperSDS-Operator의 main 함수

```go
// hypersds-operator/main.go

if err = (&controllers.CephClusterReconciler{
	Client: mgr.GetClient(),
	Log:    ctrl.Log.WithName("controllers").WithName("CephCluster"),
	Scheme: mgr.GetScheme(),
}).SetupWithManager(mgr); err != nil {
	setupLog.Error(err, "unable to create controller", "controller", "CephCluster")
	os.Exit(1)
}

// +kubebuilder:scaffold:builder

setupLog.Info("starting manager")
if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
	setupLog.Error(err, "problem running manager")
	os.Exit(1)
}
```


#### 2. Kubebuilder가 자동으로 생성해주는 controller 코드<a name="sec2"></a>

```go
// hypersds-operator/controller/cephcluster_controller.go

import (
	ctrl "sigs.k8s.io/controller-runtime"
)

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


#### 3. Builder 클래스

Builder는 controller-runtime 패키지들을 wrapping하여 builder pattern으로 controller를 생성해주는 패키지

[2번](#sec2)의 ControllerManagedBy 함수는 manager의 관리를 받는 builder 객체 주소를 return

```go
// controller-runtime/pkg/builder/controller.go

func ControllerManagedBy(m manager.Manager) *Builder {
	return &Builder{mgr: m}
}
```


즉, kubebuilder가 생성한 위 [2번](#sec2)의 SetupWithManager의 실제 코드는 아래와 같음
- Builder의 For, Watches, Completes를 수행

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
- Watch 할 input을 watch list에 append
- 나중에 호출하는 Completes 함수를 통해 이 watch list의 resource들에 Watch 호출

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


#### 4. Builder의 Complete 및 Build 함수

For, Watches, Owns 함수 등을 호출한 후 최종적으로 Completes를 호출하여 위에서 정의된 controller를 빌드함

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


#### 5. Builder의 doController<a name="sec5"></a>

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

#### 6. Controller의 New<a name="sec6"></a>

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


#### 7. Controller의 NewUnmanaged

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


#### 8. Manager의 Add

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
        cm.leaderElectionRunnables = append(cm.lea
	derElectionRunnables, r)
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

#### 9. Builder의 doWatch

[5번](#sec5)의 doController가 이렇게 다 끝나고 나면 doWatch를 수행
doWatch를 통해 controller의 CR과 watch request를 날린 모든 input들에 대해 watch를 수행

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

#### 13. Manager의 Start

위와 같은 과정으로 controller 생성 및 Watch 설정까지 마친 후 [1번](#sec1)과 같이 manager Start를 하면 add한 controller를 아래와 같이 start시킴

```
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
