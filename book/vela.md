

# KubeVela 源码阅读

[TOC]

![](/Users/zhuhuijun/go/src/github.com/ghostbaby/kubevela-cookbook/image/KubeVela.png)

## 项目介绍

- 对于平台构建者而言，KubeVela 是一个框架，能够轻松创建用户友好且高度可扩展的平台。详细地说，KubeVela 通过执行以下操作减轻了构建此类平台的复杂度：

  - 以应用为中心。KubeVela 强制采用一种**应用程序**概念作为其主要 API，并且 **所有**  KubeVela 的功能仅可满足应用程序的需求。这是通过采用[开放应用程序模型](https://github.com/oam-dev/spec)作为 KubeVela 的核心API来实现的。

  - 本地扩展。KubeVela中的应用程序由各种模块化组件（名为：服务）组成。Kubernetes 生态系统的功能可以随时通过Kubernetes CRD 注册机制作为新的工作负载类型或特征添加到 KubeVela 中。

  - 简单但可扩展的抽象机制。KubeVela 引入了一个模板引擎（支持 [CUELang](https://github.com/cuelang/cue) 等），用于从下划线的 Kubernetes 资源中提取面向用户的模式。KubeVela 提供了一组内置的抽象作为起点，平台构建者可以随时自由地对其进行修改。抽象更改将在运行时生效，无需重新编译或重新部署 KubeVela。

    有了 KubeVela，平台构建者现在终于获得了工具支持，以高信心和低周转时间设计并向其最终用户交付任何新功能。

- 对于开发人员而言，使用KubeVela构建的此类平台将使他们能够以最小的努力设计并将其应用程序发布到Kubernetes。他们只需要一个简单的应用程序定义，而不是管理少量的基础结构详细信息，而是遵循以开发人员为中心的工作流，该工作流可以轻松地与任何CI / CD管道集成。

<img src="../image/kubevela-how-it-works.png" />

##  项目组件

| 组件名称  | 功能                                                         | 入口目录                          |
| --------- | ------------------------------------------------------------ | --------------------------------- |
| core      | 该组件提供核心资源定义以及对应控制器实现                     | cmd/core/main.go                  |
| api       | 该组件提供核心资源 CURD、环境、注册中心 等资源的 Restful api | pkg/server/main/startAPIServer.go |
| cli       | 该组件提供 初始化环境、能力注册、状态查询 等命令行功能       | cmd/vela/main.go                  |
| dashboard | 该组件为前端页面代码，提供与 cli 类似的功能                  | dashboard/                        |

## 源码阅读

### 核心资源

#### 总览

| GV                    | Kind                     | 说明                                                         |
| --------------------- | ------------------------ | ------------------------------------------------------------ |
| core.oam.dev/v1alpha2 | WorkloadDefinition       | `WorkloadDefinition` 将 `Kubernetes` 内部 和 自定义 资源 在 OAM 解释器中注册，并抽象成单独负载类型，用于 OAM 统一进行调度和处理。负载类型主要针对通用负载控制器和特定负载控制器：<br />- 通用负载控制器，比如原生负载类型比如 `Deployment`、`StatefulSet`等，以及拓展负载类型比如 openKruise 中的 [`CloneSet`](https://openkruise.io/en-us/docs/cloneset.html)、[`Advanced StatefulSet`](https://openkruise.io/en-us/docs/advanced_statefulset.html)等。<br />- 特定负载控制器，比如特定组件的控制器，比如 `Flink`、`Redis`、`ES`等组件的 Operator，这类负载控制器主要用于管理对应组件的生命周期的。 |
| core.oam.dev/v1alpha2 | TraitDefinition          | `TraitDefinition` 与上述  `WorkloadDefinition`  类似，但是控制器功能有区别，`TraitDefinition` 主要用于对运维侧控制器的抽象，比如：`ManualScalerTrait`、`RouteTrait`、`AutoScalerTrait`等。<br />`ManualScalerTrait` 主要用于修改实例化 component 副本数.<br />`RouteTrait `主要用于给实例化 component 创建 ingress 规则<br />`AutoScalerTrait` 则用于给 deployment 创建 hpa 规则。 |
| core.oam.dev/v1alpha2 | ScopeDefinition          | 根据 `Component` 中的应用逻辑或共同行为划定作用域，将其分组以便于管理。<br />`ApplicationScope` 具有以下特征：<br />一个 `Component` 可能属于一个或多个 `ApplicationScope`；<br /> 有的 `ApplicationScope` 可以限定其中是否可以部署同一个 `Component` 的多个实例； <br />`ApplicationScope` 可以作为 `Component` 与基础设施的连接层，提供身份、网络或安全能力； <br />`Trait` 可以根据 `Component` 中定义的 `ApplicationScope` 来执行适当的运维特性； |
| core.oam.dev/v1alpha2 | Component                | `Component` 用于定义应用程序的基本组件，其中包含了对 Workload 的引用，一个 Component 中只能定义一个 Workload，这个 Workload 是与平台无关的，可以直接引用 Kubernetes 中的 CRD。 |
| core.oam.dev/v1alpha2 | ApplicationConfiguration | `ApplicationConfiguration` 将 `Component` 与 `Trait` 组合，定义了一个应用程序的配置，`Component` 每部署一次就会产生一个实例（`Instance`），实例是可以被升级的（包括回滚和重新部署），而每次部署和升级就会产生一次新的发布（`Release`）。 |
| core.oam.dev/v1alpha2 | ContainerizedWorkload    | ContainerizedWorkload 在原生 `Deployment`和 `Service `基础上进行抽象，只暴露 `Container`可配置选项，由控制器将 `Container` 配置渲染成 `Deployment`和 `Service `实例。该种控制器属于工作负载类型，通过 `WorkloadDefinition` 注册为 `workload`。 |
| core.oam.dev/v1alpha2 | ManualScalerTrait        | ManualScalerTrait 属于 `trait`类型的控制器，主要功能用于配置实例的副本数。 |
| core.oam.dev/v1alpha2 | HealthScope              | HealthScope 属于 `scope` 类型的控制器，主要功能用于检查对应 workload 实例化资源 status 中是否有未就绪的 pod。 |
| core.oam.dev/v1alpha2 | Application              |                                                              |
| core.oam.dev/v1alpha2 | ApplicationDeployment    |                                                              |

### core

#### main.go

##### 启动参数

| 参数                      | 类型          | 默认值                              | 说明                                                         |
| ------------------------- | ------------- | ----------------------------------- | ------------------------------------------------------------ |
| use-webhook               | bool          | false                               | 是否开启 Admission Webhook                                   |
| use-trait-injector        | bool          | false                               | 是否开启 TraitInjector                                       |
| webhook-cert-dir          | bool          | "/k8s-webhook-server/serving-certs" | Admission webhook 证书目录                                   |
| webhook-port              | int           | 9443                                | Admission webhook 监听端口                                   |
| metrics-addr              | string        | ":8080"                             | Prometheus exporter 监听地址                                 |
| enable-leader-election    | bool          | false                               | 是否开启控制器主阶段选举。<br />开启之后同一时间只允许一个控制器处于激活状态，<br />其他副本控制器将处于休眠状态。 |
| leader-election-namespace | string        | ""                                  | 定义主节点选举 configmap 所在 namespace<br />默认处于 default namespace 下 |
| log-file-path             | string        | ""                                  | 日志落盘目录                                                 |
| log-retain-date           | int           | 7                                   | 日志落盘保留时间，默认保留7天                                |
| log-compress              | bool          | true                                | 是否压缩日志                                                 |
| revision-limit            | int           | 50                                  | RevisionLimit是将要维护的最大修订版本数。 默认值为50         |
| health-addr               | string        | ":9440"                             | 健康检查接口监听地址                                         |
| apply-once-only           | bool          | false                               | 如果spec没有发生变化，不允许进行apply操作                    |
| custom-revision-hook-url  | string        | ""                                  | 是否自定义 webhook 用于返回自定义 revision 版本号            |
| disable-caps              | string        | ""                                  | 是否禁用的内置功能列表                                       |
| storage-driver            | string        | "Local"                             | Application 文件存放的存储介质，默认为本地存放               |
| informer-re-sync-interval | time.Duration | 5*time.Minute                       | 控制器全量从 informer lister 获取所有 item 的间隔            |

##### 初始化日志配置

```go
// setup logging
	var w io.Writer
	// 如果 logFilePath 已经配置
	if len(logFilePath) > 0 {
    // AddSync将io.Writer转换为WriteSyncer, 此处 io.Writer 使用 natefinch/lumberjack 包进行实例化。
    // Lumberjack是一个Go软件包，用于将日志写入滚动文件。
		w = zapcore.AddSync(&lumberjack.Logger{
      // 定义日志目录
			Filename: logFilePath,
      // 保留7天日志
			MaxAge:   logRetainDate, // days
      // 禁用日志压缩
			Compress: logCompress,
		})
	} else {
		w = os.Stdout
	}

	ctrl.SetLogger(zap.New(func(o *zap.Options) {
		o.Development = true
		o.DestWritter = w
	}))
```

##### 初始化 Manager

```go
var (
	setupLog           = ctrl.Log.WithName(kubevelaName)
	scheme             = runtime.NewScheme()
	waitSecretTimeout  = 90 * time.Second
	waitSecretInterval = 2 * time.Second
)

func init() {
  // 将 client-go 客户端集的所有类型添加到给定 scheme 中
	_ = clientgoscheme.AddToScheme(scheme)
  // 将 crd 资源添加到给定 scheme 中
	_ = crdv1.AddToScheme(scheme)
  // 将 core.oam.dev 资源添加到给定 scheme 中
	_ = oamcore.AddToScheme(scheme)
  // 将 prometheus-operator 资源添加到给定 scheme 中
	_ = monitoring.AddToScheme(scheme)
  // 将 standard.oam.dev 资源添加到给定 scheme 中
	_ = velacore.AddToScheme(scheme)
  // 将 trait-inject 资源添加到给定 scheme 中
	_ = injectorv1alpha1.AddToScheme(scheme)
  // 将 cert-manager 资源添加到给定 scheme 中
	_ = certmanager.AddToScheme(scheme)
  // 将 keda 资源添加到给定 scheme 中
	_ = kedav1alpha1.AddToScheme(scheme)
  // 将 kruise 资源添加到给定 scheme 中
  _ = kruise.AddToScheme(scheme)
	// +kubebuilder:scaffold:scheme
}

mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
  //  新建 scheme，默认可插拔
		Scheme:                  scheme,
  // 指定 prometheus exporter 监听地址及端口
		MetricsBindAddress:      metricsAddr,
  // 是否开启选主
		LeaderElection:          enableLeaderElection,
  // 指定选主 configmap 所在 namespace
		LeaderElectionNamespace: leaderElectionNamespace,
  // 指定选主 ID，LeaderElectionID确定领导者选举将用于持有领导者锁的配置映射的名称。
		LeaderElectionID:        kubevelaName,
  // 指定 webhook 监听端口
		Port:                    webhookPort,
  // 指定 webhook 证书所在目录
		CertDir:                 certDir,
  // HealthProbeBindAddress是控制器为服务健康探测而绑定的TCP地址
		HealthProbeBindAddress:  healthAddr,
  // 指定控制器全量从 informer lister 获取所有 item 的间隔，
  // SyncPeriod确定所监视资源的最低协调频率。 较低的周期将更快速地校正数据偏差，但是如果有许多监视的资源，则会降低对更改的响应速度。 仅当您知道自己在做什么时才更改此值。 如果未设置，则默认为10小时。 所有控制器的SyncPeriod之间将有10％的抖动，因此所有控制器将不会同时发送列表请求。
		SyncPeriod:              &syncPeriod,
	})
	if err != nil {
		setupLog.Error(err, "unable to create a controller manager")
		os.Exit(1)
	}
```

##### 初始化健康检查器

```go
// registerHealthChecks 主要用于创建 readiness&liveness probes
if err := registerHealthChecks(mgr); err != nil {
		setupLog.Error(err, "unable to register ready/health checks")
		os.Exit(1)
	}
```

##### 检查 capabilities

```go
const (
	// AutoscaleControllerName is the controller name of Trait autoscale
	AutoscaleControllerName = "autoscale"
	// MetricsControllerName is the controller name of Trait metrics
	MetricsControllerName = "metrics"
	// PodspecWorkloadControllerName is the controller name of Workload podsepcworkload
	PodspecWorkloadControllerName = "podspecworkload"
	// RouteControllerName is the controller name of Trait route
	RouteControllerName = "route"

	// DisableAllCaps disable all capabilities
	DisableAllCaps = "all"
	// DisableNoneCaps disable none of capabilities
	DisableNoneCaps = ""
)

// CheckDisabledCapabilities 主要功能校验 disableCaps 输入内置控制器是否合法。
// 如果 disableCaps == "" 或者 "all",则直接通过。
// 如果 disableCaps 输入多个值，通过逗号进行分割，则对字符串进行拆分，比对每个控制器是否合法
// 目前包含主要控制器如上定义的常量：autoscale、metrics、podspecworkload、route
if err := utils.CheckDisabledCapabilities(disableCaps); err != nil {
		setupLog.Error(err, "unable to get enabled capabilities")
		os.Exit(1)
	}
```

##### 初始化 Webhook

```go
var (
	setupLog           = ctrl.Log.WithName(kubevelaName)
	scheme             = runtime.NewScheme()
	waitSecretTimeout  = 90 * time.Second
	waitSecretInterval = 2 * time.Second
)

// 如果 useWebhook 为true，说明需要启动 webhook
if useWebhook {
		setupLog.Info("vela webhook enabled, will serving at :" + strconv.Itoa(webhookPort))
  // 启动 core core.oam.dev webhook
		if err = oamwebhook.Register(mgr); err != nil {
			setupLog.Error(err, "unable to setup oam runtime webhook")
			os.Exit(1)
		}
  // 启动 vela standard.oam.dev webhook
		velawebhook.Register(mgr, disableCaps)
  // waitWebhookSecretVolume 主要用于等待 webhook cert secret 创建就绪，超时时间为90s， 每两秒检查一次是否超时，如果超时则返回错误，core 启动将会推出。
		if err := waitWebhookSecretVolume(certDir, waitSecretTimeout, waitSecretInterval); err != nil {
			setupLog.Error(err, "unable to get webhook secret")
			os.Exit(1)
		}
	}
```

##### 检查 applyOnceOnly

```go
// applyOnceOnly 该特性如果打开，代表 workload 或者 trait 如果 spec 中配置没有发生变化，则不进行执行 apply 操作
switch strings.ToLower(applyOnceOnly) {
  // 如果值为 空、false、off 则关闭该特性
	case "", "false", string(oamcontroller.ApplyOnceOnlyOff):
		controllerArgs.ApplyMode = oamcontroller.ApplyOnceOnlyOff
		setupLog.Info("ApplyOnceOnly is disabled")
  // 如果值为 true、on 则打开特性
	case "true", string(oamcontroller.ApplyOnceOnlyOn):
		controllerArgs.ApplyMode = oamcontroller.ApplyOnceOnlyOn
		setupLog.Info("ApplyOnceOnly is enabled, that means workload or trait only apply once if no spec change even they are changed by others")
	case string(oamcontroller.ApplyOnceOnlyForce):
		controllerArgs.ApplyMode = oamcontroller.ApplyOnceOnlyForce
		setupLog.Info("ApplyOnceOnlyForce is enabled, that means workload or trait only apply once if no spec change even they are changed or deleted by others")
	default:
		setupLog.Error(fmt.Errorf("invalid apply-once-only value: %s", applyOnceOnly),
			"unable to setup the vela core controller",
			"valid apply-once-only value:", "on/off/force, by default it's off")
		os.Exit(1)
	}
```

##### 装载控制器

```go
	// 装载 oam core 控制器，包含：applicationconfiguration、containerizedworkload、manualscalertrait、healthscope、application、applicationdeployment
	if err = oamv1alpha2.Setup(mgr, controllerArgs, logging.NewLogrLogger(setupLog)); err != nil {
		setupLog.Error(err, "unable to setup the oam core controller")
		os.Exit(1)
	}

	// 装载 vela core 控制器
	// 如果不禁用任何控制器，将装载如下控制器：metrics、podspecworkload、routes、autoscaler
	// 如果禁用所有，将不装载任何控制器
	// 其他情况，需要校验 disableCaps 输入值，如果存在对于控制，则会跳过装载
	if err = velacontroller.Setup(mgr, disableCaps); err != nil {
		setupLog.Error(err, "unable to setup the vela core controller")
		os.Exit(1)
	}
```

##### 检查存储环境变量

```go
// 如果 STORAGE_DRIVER 环境变量没有设置，则设置 storageDriver 值
if driver := os.Getenv(system.StorageDriverEnv); len(driver) == 0 {
		// first use system environment,
		err := os.Setenv(system.StorageDriverEnv, storageDriver)
		if err != nil {
			setupLog.Error(err, "unable to setup the vela core controller")
			os.Exit(1)
		}
	}
	setupLog.Info("use storage driver", "storageDriver", os.Getenv(system.StorageDriverEnv))

```

##### 初始化 Trait 注入

```go
// TraitInjector，是 kubernetes admission webhook，主要用于注入 env 和 volume 到workload。
// 目前将服务绑定进行资源抽象，抽象为 ServiceBinding，将 secret 或者 pvc 绑定到指定 deployment 或者 sts 中。
if useTraitInjector {
		// register all service injectors
		plugin.RegisterTargetInjectors(injector.Defaults()...)

		tiWebhook := &injectorcontroller.ServiceBindingReconciler{
			Client:   mgr.GetClient(),
			Log:      ctrl.Log.WithName("controllers").WithName("ServiceBinding"),
			Scheme:   mgr.GetScheme(),
			Recorder: mgr.GetEventRecorderFor("servicebinding"),
		}
		if err = (tiWebhook).SetupWithManager(mgr); err != nil {
			setupLog.Error(err, "unable to create controller", "controller", "ServiceBinding")
			os.Exit(1)
		}
		// this has hard coded requirement "./ssl/service-injector.pem", "./ssl/service-injector.key"
		go tiWebhook.ServeAdmission()
	}
```

##### 启动 vela 控制管理器

```go
	setupLog.Info("starting the vela controller manager")

	if err := mgr.Start(makeSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
	setupLog.Info("program safely stops...")
```

#### Webhook

##### oam-core webhook

###### 总览

| name                                               | path                                                        |
| -------------------------------------------------- | ----------------------------------------------------------- |
| application.RegisterValidatingHandler              | /validating-core-oam-dev-v1alpha2-applications              |
| applicationconfiguration.RegisterValidatingHandler | /validating-core-oam-dev-v1alpha2-applicationconfigurations |
| traitdefinition.RegisterValidatingHandler          | /validating-core-oam-dev-v1alpha2-traitdefinitions          |
| applicationconfiguration.RegisterMutatingHandler   | /mutating-core-oam-dev-v1alpha2-applicationconfigurations   |
| applicationdeployment.RegisterMutatingHandler      | /mutating-core-oam-dev-v1alpha2-applicationdeployments      |
| component.RegisterMutatingHandler                  | /mutating-core-oam-dev-v1alpha2-components                  |
| component.RegisterValidatingHandler                | /validating-core-oam-dev-v1alpha2-components                |

###### Register

```go
// Register will be called in main and register all validation handlers
func Register(mgr manager.Manager) error {
	if err := application.RegisterValidatingHandler(mgr); err != nil {
		return err
	}
	if err := applicationconfiguration.RegisterValidatingHandler(mgr); err != nil {
		return err
	}
	if err := traitdefinition.RegisterValidatingHandler(mgr); err != nil {
		return err
	}
	applicationconfiguration.RegisterMutatingHandler(mgr)
	applicationdeployment.RegisterMutatingHandler(mgr)
	if err := component.RegisterMutatingHandler(mgr); err != nil {
		return err
	}
	component.RegisterValidatingHandler(mgr)
	return nil
}

```

Register，主要用于注册所有 core.oam.dev 下 wehbook 控制器，主要包含 validation 和 mutating两种控制器，包括如下资源：

- validation
  - application
  - applicationconfiguration
  - traitdefinition
  - component
- mutating
  - applicationconfiguration
  - applicationdeployment
  - component

######validation/application

- 入口

```go
// RegisterValidatingHandler will regsiter application validate handler to the webhook
func RegisterValidatingHandler(mgr manager.Manager) error {
  // 创建 discoverymapper ，这个是一个接口类型，主要用于根据 GVK 更新和发现 Kubernetes 资源
	mapper, err := discoverymapper.New(mgr.GetConfig())
	if err != nil {
		return err
	}
	server := mgr.GetWebhookServer()
  // webhook 服务path
	server.Register("/validating-core-oam-dev-v1alpha2-applications", &webhook.Admission{Handler: &ValidatingHandler{dm: mapper}})
	return nil
}

```



- 核心逻辑

```go
// Handle validate Application Spec here
func (h *ValidatingHandler) Handle(ctx context.Context, req admission.Request) admission.Response {
	app := &v1alpha2.Application{}
  // 将 admission 请求解析成 Application
	if err := h.Decoder.Decode(req, app); err != nil {
		return admission.Errored(http.StatusBadRequest, err)
	}

	switch req.Operation {
    // 如果当前请求类型为 Create，则执行 ValidateCreate()，该方法主要用于将请求中的 application 资源，转换成 Appfile 对象
    // todo：描述Appfile对象作用
	case admissionv1beta1.Create:
		if allErrs := h.ValidateCreate(app); len(allErrs) > 0 {
			return admission.Errored(http.StatusUnprocessableEntity, allErrs.ToAggregate())
		}
    // 如果当前请求类型是 Update，先获取 current 对象，再执行 ValidateUpdate()。
    // ValidateUpdate() ，先执行上面 ValidateCreate()，然后分别获取 current 和 expect 对象的 annotation ，检查 app.oam.dev/rollout-template 是否存在，如果 current 不存在但是 expect 存在，则返回 error 拒绝请求。
    // 主要是不允许给已经存在的对象添加 app.oam.dev/rollout-template 标签。    
	case admissionv1beta1.Update:
		oldApp := &v1alpha2.Application{}
		if err := h.Decoder.DecodeRaw(req.AdmissionRequest.OldObject, oldApp); err != nil {
			return admission.Errored(http.StatusBadRequest, err)
		}

		if allErrs := h.ValidateUpdate(app, oldApp); len(allErrs) > 0 {
			return admission.Errored(http.StatusUnprocessableEntity, allErrs.ToAggregate())
		}
	default:
		// Do nothing for DELETE and CONNECT
	}
	return admission.ValidationResponse(true, "")
}
```

###### validation/applicationconfiguration(ac)

```go
// RegisterValidatingHandler will register application configuration validation to webhook
func RegisterValidatingHandler(mgr manager.Manager) error {
	server := mgr.GetWebhookServer()
	mapper, err := discoverymapper.New(mgr.GetConfig())
	if err != nil {
		return err
	}
	server.Register("/validating-core-oam-dev-v1alpha2-applicationconfigurations", &webhook.Admission{Handler: &ValidatingHandler{
		Mapper: mapper,
		Validators: []AppConfigValidator{
      // ValidateRevisionNameFn 校验 revisionName 和 componentName 是否同时配置，如果两个配置同时有则拒绝请求.
			AppConfigValidateFunc(ValidateRevisionNameFn),
      // ValidateWorkloadNameForVersioningFn 校验当开启 version-enabled 时候的 workload 名称
      // 检查 ac component 中有没有配置 workload name ，如果没有配置或者不为空都拒绝请求。
			AppConfigValidateFunc(ValidateWorkloadNameForVersioningFn),
      // 检查 ac component 中配置的 trait 是否允许被应用在给定 workload中，如果不允许则拒绝请求。
      // 主要检查逻辑如下：
      // 1. 如果指定 trait 配置中 traitDefinition.Spec.AppliesToWorkloads 字段未配置，则说明改 trait 可以应用于所有 workload，可以跳过改校验。
      // 2. 如果 AppliesToWorkloads 内等于或者包含 * 字符，则跳过校验。
      // 3. 如果 AppliesToWorkloads 内包含当前 workload crd 名字，则跳过校验。
      // 4. 如果上述过滤器均不匹配，则说明该 trait 不能应用于当前 workload，将拒绝当前请求。
			AppConfigValidateFunc(ValidateTraitAppliableToWorkloadFn),
      // ValidateTraitConflictFn 用于校验是否有逻辑冲突的 trait 应用于同一个 workload 。
      // 主要检查逻辑如下：
      // 1. 采集所有 component 中定义的 trait Spec.ConflictsWith 冲突规则，并存储到 map，key 为 trait 名称，value 为 冲突对象。
      // 2. 如果该 component 中定义的 trait ConflictsWith 字段为空，则说明所有 trait 没有冲突，直接通过当前校验。
      // 3. 循环上述map，如果有规则等于 “*”，说明当前 trait 只能单独应用于 workload，如果当前 workload 配置 trait 个数不唯一，则会拒绝当前请求。
      // 4. 循环上述map，遍历 component 中配置的 trait，检查 API group、CRD name、trait definition name、labelSelector 是否于当前规则冲突，有一项冲突将拒绝当前请求。
			AppConfigValidateFunc(ValidateTraitConflictFn),
			// TODO(wonderflow): Add more validation logic here.
		},
	}})
	return nil
}
```

###### validation/traitdefinition

```go
// 如果 Spec.Reference.Name 没有配置，且 Spec.Template 也为空时，则拒绝当前请求。
func ValidateDefinitionReference(_ context.Context, td v1alpha2.TraitDefinition) error {
	if len(td.Spec.Reference.Name) > 0 {
		return nil
	}
	tmp, err := util.NewTemplate(td.Spec.Template, td.Spec.Status, td.Spec.Extension)
	if err != nil {
		return errors.Wrap(err, errValidateDefRef)
	}
	if len(tmp.TemplateStr) == 0 {
		return errors.New(failInfoDefRefOmitted)
	}
	return nil
}
```

###### validation/component

```go
// ValidateComponentObject validates the Component on creation
func ValidateComponentObject(obj *v1alpha2.Component) field.ErrorList {
	validatelog.Info("validate component", "name", obj.Name)
  // 校验 ObjectMeta 配置，包括 namespace、name、generateName、finalizers等，如果校验失败则拒绝请求
	allErrs := apimachineryvalidation.ValidateObjectMeta(&obj.ObjectMeta, true,
		apimachineryvalidation.NameIsDNSSubdomain, field.NewPath("metadata"))
	fldPath := field.NewPath("spec")
	var content map[string]interface{}
	if err := json.Unmarshal(obj.Spec.Workload.Raw, &content); err != nil {
		allErrs = append(allErrs, field.Invalid(fldPath.Child("workload"), string(obj.Spec.Workload.Raw),
			"the workload is malformat"))
		return allErrs
	}
	if content[TypeField] != nil {
		allErrs = append(allErrs, field.Invalid(fldPath.Child("workload"), string(obj.Spec.Workload.Raw),
			"the workload contains type info"))
	}
	workload := unstructured.Unstructured{
		Object: content,
	}
  // 校验 workload 是否配置 gvk 信息，如果配置信息为空则拒绝请求。
	if len(workload.GetAPIVersion()) == 0 || len(workload.GetKind()) == 0 {
		allErrs = append(allErrs, field.Invalid(fldPath.Child("workload"), content,
			fmt.Sprintf("the workload data missing GVK, api = %s, kind = %s,", workload.GetAPIVersion(), workload.GetKind())))
	}
	return allErrs
}
```

###### mutating/applicationconfiguration(ac)

```go
func (h *MutatingHandler) mutateTrait(content map[string]interface{}, compName string) ([]byte, bool, error) {
	// 如果 component 中 trait 项为空，则直接返回 
  if content[TraitTypeField] == nil {
		return nil, false, nil
	}
  // 获取 TraitDefinition 名称，用于后面获取 crd 信息 
	traitType, ok := content[TraitTypeField].(string)
	if !ok {
		return nil, false, fmt.Errorf("name of trait should be string instead of %s", reflect.TypeOf(content[TraitTypeField]))
	}
	mutatelog.Info("the trait refers to traitDefinition by name", "compName", compName, "trait name", traitType)
	// Fetch the corresponding traitDefinition CR, the traitDefinition crd is cluster scoped
  // 获取 component 指定的 TraitDefinition 对象
	traitDefinition := &v1alpha2.TraitDefinition{}
	if err := h.Client.Get(context.TODO(), types.NamespacedName{Name: traitType}, traitDefinition); err != nil {
		return nil, false, err
	}
	// fetch the CRDs definition
  // 获取 TraitDefinition 对象依赖的 CRD
	customResourceDefinition := &crdv1.CustomResourceDefinition{}
	if err := h.Client.Get(context.TODO(), types.NamespacedName{Name: traitDefinition.Spec.Reference.Name}, customResourceDefinition); err != nil {
		return nil, false, err
	}
  // 清除 map 中 TraitTypeField 字段，用于后续 trait 对象的重建
	// reconstruct the trait CR
	delete(content, TraitTypeField)

	if content[TraitSpecField] != nil {
		content["spec"] = content[TraitSpecField]
		delete(content, TraitSpecField)
	}

	trait := unstructured.Unstructured{
		Object: content,
	}
	// find out the GVK from the CRD definition and set
  // 从 CRD 中获取 TraitDefinition 中定义 trait 对应的 GVK 信息
	apiVersion := metav1.GroupVersion{
		Group:   customResourceDefinition.Spec.Group,
		Version: customResourceDefinition.Spec.Versions[0].Name,
	}.String()
  // 为新建 trait 对象设置 apiVersion
	trait.SetAPIVersion(apiVersion)
  // 设置 kind
	trait.SetKind(customResourceDefinition.Spec.Names.Kind)
	mutatelog.Info("Set the trait GVK", "trait api version", trait.GetAPIVersion(), "trait Kind", trait.GetKind())
	// add traitType label
  // 设置 trait.oam.dev/type 标签
	trait.SetLabels(util.MergeMapOverrideWithDst(trait.GetLabels(), map[string]string{oam.TraitTypeLabel: traitType}))
	// copy back the object
	rawBye, err := json.Marshal(trait.Object)
	if err != nil {
		return nil, false, err
	}
	return rawBye, true, nil
}
```

###### mutating/applicationdeployment(ad)

```go
// DefaultApplicationDeployment will set the default value for the ApplicationDeployment
func DefaultApplicationDeployment(obj *v1alpha2.ApplicationDeployment) {
	klog.InfoS("default", "name", obj.Name)
  // 设置 obj.Spec.RevertOnDelete 默认值 false
	if obj.Spec.RevertOnDelete == nil {
		klog.V(common.LogDebug).Info("default RevertOnDelete as false")
		obj.Spec.RevertOnDelete = pointer.BoolPtr(false)
	}

	// default rollout plan
  // 设置滚动升级计划默认值
	rollout.DefaultRolloutPlan(&obj.Spec.RolloutPlan)
}
```

```go
// DefaultRolloutPlan set the default values for a rollout plan
// This is called by the mutation webhooks and before the validators
// DefaultRolloutPlan 主要用于设置滚动升级计划默认值，该方法将会在上面 mutation webhooks 调用
func DefaultRolloutPlan(rollout *v1alpha1.RolloutPlan) {
  // 当 需要滚动副本数、滚动批次数 两个值不为空，但是 每批滚动副本数 没有设置，则由该函数补全。
	if rollout.TargetSize != nil && rollout.NumBatches != nil && rollout.RolloutBatches == nil {
		// create the rollout  batch based on the total size and num batches if it's not set
		// leave it for the validator to valiate if they are both set
		numBatches := int(*rollout.NumBatches)
		totalSize := int(*rollout.TargetSize)
		// create the batch array
		rollout.RolloutBatches = make([]v1alpha1.RolloutBatch, int(*rollout.NumBatches))
    // 计算出每次平均需要滚动副本数
		avg := intstr.FromInt(totalSize / numBatches)
		total := 0
		for i := 0; i < numBatches-1; i++ {
      // 将每批滚动副本数写入RolloutBatches
			rollout.RolloutBatches[i].Replicas = avg
      // 获取当前总共计算过的副本数
			total += avg.IntValue()
		}
		// fill out the last batch
    // 设置最后一批分批滚动副本数
		rollout.RolloutBatches[numBatches-1].Replicas = intstr.FromInt(totalSize - total)
	}
}
```

###### mutating/component

```go
// Mutate sets all the default value for the Component
func (h *MutatingHandler) Mutate(obj *v1alpha2.Component) error {
	mutatelog.Info("mutate", "name", obj.Name)
	var content map[string]interface{}
	if err := json.Unmarshal(obj.Spec.Workload.Raw, &content); err != nil {
		return err
	}
	if content[TypeField] != nil {
    // 获取 component 定义的 WorkloadDefinition 名称
		workloadType, ok := content[TypeField].(string)
		if !ok {
			return fmt.Errorf("workload content has an unknown type field")
		}
		mutatelog.Info("the component refers to workoadDefinition by type", "name", obj.Name, "workload type", workloadType)
		// Fetch the corresponding workloadDefinition CR, the workloadDefinition crd is cluster scoped
    // 获取 WorkloadDefinition 对象
		workloadDefinition := &v1alpha2.WorkloadDefinition{}
		if err := h.Client.Get(context.TODO(), types.NamespacedName{Name: workloadType}, workloadDefinition); err != nil {
			return err
		}
    // 获取 WorkloadDefinition 对应的 GVK 信息， 主要通过获取 WorkloadDefinition 定义的 CRD 资源，然后获取 CRD 中定义的 GVK 信息
		gvk, err := util.GetGVKFromDefinition(h.Mapper, workloadDefinition.Spec.Reference)
		if err != nil {
			return err
		}
    // 清洗原始对象中的 name 字段
		// reconstruct the workload CR
		delete(content, TypeField)
    // 将原始 map 对象转换为 unstructured 结构
		workload := unstructured.Unstructured{
			Object: content,
		}
		// find out the GVK from the CRD definition and set
		apiVersion := metav1.GroupVersion{
			Group:   gvk.Group,
			Version: gvk.Version,
		}.String()
    // 设置 apiVersion
		workload.SetAPIVersion(apiVersion)
    // 设置 kind
		workload.SetKind(gvk.Kind)
		mutatelog.Info("Set the component workload GVK", "workload api version", workload.GetAPIVersion(), "workload Kind", workload.GetKind())
		// copy namespace/label/annotation to the workload and add workloadType label
    // 设置 namespace
		workload.SetNamespace(obj.GetNamespace())
    // 设置 label
		workload.SetLabels(util.MergeMapOverrideWithDst(obj.GetLabels(), map[string]string{oam.WorkloadTypeLabel: workloadType}))
    // 设置 annotation
		workload.SetAnnotations(obj.GetAnnotations())
    // 上述操作总结，将 component 定义的特性传递给 workload 定义
		// copy back the object
		rawBye, err := json.Marshal(workload.Object)
		if err != nil {
			return err
		}
		obj.Spec.Workload.Raw = rawBye
	}

	return nil
}
```

##### vela standard.oam.dev webhook

###### 总揽

| name                       | path                                                |
| -------------------------- | --------------------------------------------------- |
| MetricsTrait.Validating    | /validate-standard-oam-dev-v1alpha1-metricstrait    |
| MetricsTrait.Mutating      | /mutate-standard-oam-dev-v1alpha1-metricstrait      |
| PodSpecWorkload.Validating | /validate-standard-oam-dev-v1alpha1-podspecworkload |
| PodSpecWorkload.Mutating   | /mutate-standard-oam-dev-v1alpha1-podspecworkload   |

###### validation/metricstrait

```go
// ValidateCreate validates the metricsTrait on creation
func ValidateCreate(r *v1alpha1.MetricsTrait) field.ErrorList {
	validatelog.Info("validate create", "name", r.Name)
  // 校验 ObjectMeta 配置，包括 namespace、name、generateName、finalizers等，如果校验失败则拒绝请求
	allErrs := apimachineryvalidation.ValidateObjectMeta(&r.ObjectMeta, true,
		apimachineryvalidation.NameIsDNSSubdomain, field.NewPath("metadata"))
	fldPath := field.NewPath("spec")
  // 校验 Spec.ScrapeService.Format 数据采集格式是否为 prometheus，如果是则通过，不是则添加错误信息
	if r.Spec.ScrapeService.Format != SupportedFormat {
		allErrs = append(allErrs, field.Invalid(fldPath.Child("ScrapeService.Format"), r.Spec.ScrapeService.Format,
			fmt.Sprintf("the data format `%s` is not supported", r.Spec.ScrapeService.Format)))
	}
  // 校验 Spec.ScrapeService.Scheme 是否为 http，如果是则通过，不是则添加错误信息
	if r.Spec.ScrapeService.Scheme != SupportedScheme {
		allErrs = append(allErrs, field.Invalid(fldPath.Child("ScrapeService.Format"), r.Spec.ScrapeService.Scheme,
			fmt.Sprintf("the scheme `%s` is not supported", r.Spec.ScrapeService.Scheme)))
	}
  // 总结，目前metricstrait只支持 http 模式下的 prometheus exporter
	return allErrs
}
```

###### mutating/metricstrait

```go
// DefaultMetrics sets all the default value for the metricsTrait
func DefaultMetrics(obj *v1alpha1.MetricsTrait) {
	mutatelog.Info("default", "name", obj.Name)
  // 如果 Spec.ScrapeService.Format 指标格式未配置，则配置为默认值，即 prometheus
	if len(obj.Spec.ScrapeService.Format) == 0 {
		mutatelog.Info("default format as prometheus")
		obj.Spec.ScrapeService.Format = SupportedFormat
	}
  // 如果 Spec.ScrapeService.Path 指标路径未配置，则配置为默认值，即 /metrics
	if len(obj.Spec.ScrapeService.Path) == 0 {
		mutatelog.Info("default path as /metrics")
		obj.Spec.ScrapeService.Path = DefaultMetricsPath
	}
  // 如果 Spec.ScrapeService.Scheme URL scheme 为配置，则配置为默认值，即 http
	if len(obj.Spec.ScrapeService.Scheme) == 0 {
		mutatelog.Info("default scheme as http")
		obj.Spec.ScrapeService.Scheme = SupportedScheme
	}
  // 如果 指标采集开关没有配置，则配置默认值，默认开启
	if obj.Spec.ScrapeService.Enabled == nil {
		mutatelog.Info("default enabled as true")
		obj.Spec.ScrapeService.Enabled = pointer.BoolPtr(true)
	}
}
```

###### validation/podspecworkload

```go
// ValidateCreate validates the PodSpecWorkload on creation
func ValidateCreate(r *v1alpha1.PodSpecWorkload) field.ErrorList {
	validatelog.Info("validate create", "name", r.Name)
  // 校验 ObjectMeta 配置，包括 namespace、name、generateName、finalizers等，如果校验失败则拒绝请求
	allErrs := apimachineryvalidation.ValidateObjectMeta(&r.ObjectMeta, true,
		apimachineryvalidation.NameIsDNSSubdomain, field.NewPath("metadata"))

	fldPath := field.NewPath("spec")
  // 校验 spec.replicas 配置数值是否合法
	allErrs = append(allErrs, apimachineryvalidation.ValidateNonnegativeField(int64(*r.Spec.Replicas),
		fldPath.Child("Replicas"))...)

	fldPath = fldPath.Child("podSpec")
	spec := r.Spec.PodSpec
  // 如果 Containers 为空，则拒绝该请求
	if len(spec.Containers) == 0 {
		allErrs = append(allErrs, field.Invalid(fldPath.Child("Containers"), spec.Containers,
			"You need at least one container"))
	}
	return allErrs
}
```

###### mutating/podspecworkload

```go
// DefaultPodSpecWorkload will set the default value for the PodSpecWorkload
func DefaultPodSpecWorkload(obj *v1alpha1.PodSpecWorkload) {
	mutatelog.Info("default", "name", obj.Name)
  // 设置默认副本数为 1
	if obj.Spec.Replicas == nil {
		mutatelog.Info("default replicas as 1")
		obj.Spec.Replicas = pointer.Int32Ptr(1)
	}
}
```

#### Setup

##### oam-core

##### 总揽

| GV                    | Kind                     | 说明 |
| --------------------- | ------------------------ | ---- |
| core.oam.dev/v1alpha2 | ApplicationConfiguration |      |
| core.oam.dev/v1alpha2 | ContainerizedWorkload    |      |
| core.oam.dev/v1alpha2 | ManualScalerTrait        |      |
| core.oam.dev/v1alpha2 | HealthScope              |      |
| core.oam.dev/v1alpha2 | Application              |      |
| core.oam.dev/v1alpha2 | ApplicationDeployment    |      |

```go
// Setup workload controllers.
func Setup(mgr ctrl.Manager, args controller.Args, l logging.Logger) error {
	for _, setup := range []func(ctrl.Manager, controller.Args, logging.Logger) error{
		applicationconfiguration.Setup,
		containerizedworkload.Setup,
    manualscalertrait.Setup, 
    healthscope.Setup,
		application.Setup, 
    applicationdeployment.Setup,
	} {
		if err := setup(mgr, args, l); err != nil {
			return err
		}
	}
	return nil
}
```

#####ApplicationConfiguration(ac)

ApplicationConfiguration，该控制器包含对如下CRD的操作：

- ApplicationConfiguration 
- WorkloadDefinition
- TraitDefinition 
- ScopeDefinition 
- Component

该控制主要逻辑如下：

1. 消息入队，从 event 队列中获取待处理的 ac event 对象；
2. 获取对象，从 k8s 中获取当前待处理 ac 对象的终态声明信息；
3. 执行拦截器，判断当前 ac 对象是否处于待删除状态，如果处于则执行拦截器，从 scope 中删除当前 ac 对象，如果删除成功直接退出当前处理流程；
4. 执行前置操作，此处一般用于执行外部 webhook，做一些前置校验或者通知操作；
5. 渲染资源，将 application 中 component 配置渲染成内部 ` workload`  对象、以及依赖关系；
6. 提交资源，将上一步渲染出来的 `workload` 对象提交创建，包括：workload、trait、scope、以及各 workload 之前的依赖关系；
7. 执行回收，对 ac 渲染出来的所有对象执行垃圾回收。

#####Application(app)

ApplicationConfiguration，该控制器包含对如下CRD的操作：

- Application

该控制主要逻辑如下：

1. 消息入队，从 event 队列中获取待处理的 app event 对象；
2. 获取对象，从 k8s 中获取当前待处理 app 对象的终态声明信息；
3. 渲染资源，从 appfile 转换成 ac 和 component，主要逻辑如下：
   -  从 ac 中拿到所有 workload，然后从 WorkloadDefinition 和 TraitDefinition 中获取对应对象
   -  从 Definition 中拿到 cue 定义的模版部分，将 parameter 部分数据渲染到 outputField中完成数据渲染，
   -  将 workload 渲染出来的数据构建成 Component 对象，将 trait、scope 渲染出来的数据构建 acCompoent
4. 提交资源，创建上面渲染出来的 ac 和 compoent 组件；

#### ApplicationConfiguration(ac)

##### CRD

```go
type ApplicationConfiguration struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ApplicationConfigurationSpec   `json:"spec,omitempty"`
	Status ApplicationConfigurationStatus `json:"status,omitempty"`
}

type ApplicationConfigurationSpec struct {
	// Components 用于定义 ac 中需要实例化的组件列表
	Components []ApplicationConfigurationComponent `json:"components"`
}

// 用于定义 ac 中单个组件
type ApplicationConfigurationComponent struct {
	// ComponentName，用于定义组件名称，组件最新版本将与 ac 进行绑定，如果该字段发生变化，ac 控制器将会将原先作用于该组件的 trait，迁移到最新的组件上。该字段于下面 RevisionName 是互斥的。可选参数。
	ComponentName string `json:"componentName,omitempty"`

	// RevisionName，用于将特定版本的组件与 ac 进行绑定，该字段与上面 ComponentName 是互斥的。可选参数
	RevisionName string `json:"revisionName,omitempty"`

	// DataOutputs，用于定义该组件输出信息列表，比如该组件为redis，这里可以配置redis的连接信息和用户信息，用于其他资源使用。必需氨参数
	DataOutputs []DataOutput `json:"dataOutputs,omitempty"`

	// DataOutputs，用于定义该组件启动依赖的数据，比如该组件为前端应用，需要连接数据库或者缓冲，则可以通过配置该配置，传入启动所需数据
	DataInputs []DataInput `json:"dataInputs,omitempty"`

	// ParameterValues，用于定义实例化 workload 所需要必要参数，类似 helm 中的 value.yaml，可选参数
	ParameterValues []ComponentParameterValue `json:"parameterValues,omitempty"`

	// Traits，用于定义需要生效在组件上的运维特性
	Traits []ComponentTrait `json:"traits,omitempty"`

	// Scopes，用于定义组件所属 scope
	Scopes []ComponentScope `json:"scopes,omitempty"`
}

// DataOutput specifies a data output source from an object.
type DataOutput struct {
	// 数据输出时创建资源的名称，需要在 ac 范围内唯一
	Name string `json:"name,omitempty"`

	// 输出数据从哪里获取，一般是从 status 中拿，比如数据库连接信息
	FieldPath string `json:"fieldPath,omitempty"`

	// 用于指定判定当前组件是否就绪的标识位，如果该项为空，则只会检查 FieldPath 中定义的字段是否为空
	Conditions []ConditionRequirement `json:"conditions,omitempty"`

  // OutputStore，用于定义 Operations 产生的中间数据存储的信息
	OutputStore StoreReference `json:"outputStore,omitempty"`
}


// StoreReference specifies the referenced object in DataOutput or DataInput
type StoreReference struct {
  // 数据存储介质，一般是 configmap
	runtimev1alpha1.TypedReference `json:",inline"`
	// 数据处理操作，主要用于描述 数据从哪来，存哪里
	Operations []DataOperation `json:"operations,omitempty"`
}

// DataOperation defines the specific operation for data
type DataOperation struct {
  // 用于指定数据操作数据类型，这里目前只支持 jsonPath
	Type string `json:"type"`
	// 定义数据操作方法，目前支持 新增、删除、替换
	Operator DataOperator `json:"op"`
	// 用于定义存储于介质的路径，类似 spec.data，spec 下没有其他key的情况可以直接配置本字段
	ToFieldPath string `json:"toFieldPath"`
	// 用于定位value的key，上述情况 spec.data 下有多个key，就需要指定特殊字段
	ToDataPath string `json:"toDataPath,omitempty"`
	// +optional
	// 用于定义数据存储值，这里是直接设置值
	// 该字段与下面 ValueFrom 冲突
	Value string `json:"value,omitempty"`
	// +optional
	// 用于定义数据存储值获取的路径，这里需要从指定目录获取
	ValueFrom  ValueFrom              `json:"valueFrom,omitempty"`
	Conditions []ConditionRequirement `json:"conditions,omitempty"`
}
```



##### Setup

```go
// path:pkg/controller/core.oam.dev/v1alpha2/applicationconfiguration/applicationconfiguration.go
// Setup adds a controller that reconciles ApplicationConfigurations.
func Setup(mgr ctrl.Manager, args core.Args, l logging.Logger) error {
  // 初始化 discover client
	dm, err := discoverymapper.New(mgr.GetConfig())
	if err != nil {
		return fmt.Errorf("create discovery dm fail %w", err)
	}
	name := "oam/" + strings.ToLower(v1alpha2.ApplicationConfigurationGroupKind)

  // 新建控制器 builder ，该 builder 将由提供的 manager 启动
	return ctrl.NewControllerManagedBy(mgr).
  // 为新建控制器添加名字，该名字将被用于 prometheus metrics，所以该名字需要符合prometheus key的命名规则（仅支持下划线、字母、数字）
		Named(name).
  // For 用于定义`需要 reconciled` 的资源，即需要watch的资源，这里说明需要 watch ApplicationConfiguration的事件并进行 reconcile
		For(&v1alpha2.ApplicationConfiguration{}).
  // Watches 功能类似 For, 此处需要手动构建 handler，此处的 ComponentHandler 针对事件的不同类型，实现了 EventHandler 接口的方法，包括 create、update、delete、generic
		Watches(&source.Kind{Type: &v1alpha2.Component{}}, &ComponentHandler{
			Client:                mgr.GetClient(),
			Logger:                l,
			RevisionLimit:         args.RevisionLimit,
			CustomRevisionHookURL: args.CustomRevisionHookURL,
		}).
  // 初始化 Reconciler ， 主要添加 recorder、控制器名字、apply mode
		Complete(NewReconciler(mgr, dm,
			l.WithValues("controller", name),
			WithRecorder(event.NewAPIRecorder(mgr.GetEventRecorderFor(name))),
			WithApplyOnceOnlyMode(args.ApplyMode)))
}
```

>**如何监听多个资源事件**
>
>日常开发 operator 过程中，都会遇到多层资源创建的情况，比如 redis- operator 资源创建情况 CRD----> Deployment -----> Pod，这里控制器就需要监听 redis CRD 、Deployment 、Pod 的事件，针对上述多个事件进行逻辑处理。
>
>针对上述需求 controller-runtime 提供了三种方式进行事件监听：For()、Owns()、Watch()
>
>- For()，提供通用的事件监听方法，提供 EnqueueRequestForObject 方法来进行事件过滤，会监听关联资源的所有事件。
>- Owns()，提供For中定义资源的关联资源事件监听方法，比如上述 redis CRD 下面创建 Deployment 资源，创建的时候可以设置 Deployment 的 OnwerReference ，进行级联绑定资源的事件也会被监听。
>- Watch()，提供自定义事件过滤器的监听方法，前面两种方法都是基于 Watch 来实现的，可以自定义事件过滤策略。

##### NewReconciler

```go
// An OAMApplicationReconciler reconciles OAM ApplicationConfigurations by rendering and
// instantiating their Components and Traits.
type OAMApplicationReconciler struct {
  // k8s client
	client            client.Client
  // component 对象生成器，主要用于将 ac 中 component 转换成 workload 和 trait
	components        ComponentRenderer
  // workload 和 trait 资源管理器，用于创建和删除前置处理
	workloads         WorkloadApplicator
  // 用于判定资源是否符合回收条件
	gc                GarbageCollector
  // 已经注册过的 scheme 对象
	scheme            *runtime.Scheme
  // 日志对象
	log               logging.Logger
  // record 对象，用于在自定义资源中添加event
	record            event.Recorder
  // 前置控制器
	preHooks          map[string]ControllerHooks
  // 后置控制器
	postHooks         map[string]ControllerHooks
  // ApplyOnceOnly 特性开关
	applyOnceOnlyMode core.ApplyOnceOnlyMode
}
```

##### Reconcile

```go
// NOTE(negz): We don't validate anything against their definitions at the
// controller level. We assume this will be done by validating admission
// webhooks.

// Reconcile an OAM ApplicationConfigurations by rendering and instantiating its
// Components and Traits.
func (r *OAMApplicationReconciler) Reconcile(req reconcile.Request) (result reconcile.Result, returnErr error) {
	log := r.log.WithValues("request", req)
	log.Debug("Reconciling")

  // 设置带超时的上下文，默认1分钟超时
	ctx, cancel := context.WithTimeout(context.Background(), reconcileTimeout)
	defer cancel()

  // 获取当前事件队列中 ac 的资源对象
	ac := &v1alpha2.ApplicationConfiguration{}
	if err := r.client.Get(ctx, req.NamespacedName, ac); err != nil {
		if apierrors.IsNotFound(err) {
			// stop processing this resource
      // 如果不存在则退出本次主流程
			return ctrl.Result{}, nil
		}
		return reconcile.Result{}, errors.Wrap(err, errGetAppConfig)
	}
  // 克隆当前 ac 对象，用于后续状态更新用
	acPatch := ac.DeepCopy()
  // 如果当前 ac 对象没有被删除
	if ac.ObjectMeta.DeletionTimestamp.IsZero() {
    // 则注册 Finalizers ，此处主要是删除拦截配置了 scope 的 ac，需要先确保资源 从 scope reference 中删除。
		if registerFinalizers(ac) {
			log.Debug("Register new finalizers", "finalizers", ac.ObjectMeta.Finalizers)
			return reconcile.Result{}, errors.Wrap(r.client.Update(ctx, ac), errUpdateAppConfigStatus)
		}
	} else {
    // 如果当前 ac 对象已经被标记为需要被删除
    // 则需要从 scope 中删除该对象
		if err := r.workloads.Finalize(ctx, ac); err != nil {
			log.Debug("Failed to finalize workloads", "workloads status", ac.Status.Workloads,
				"error", err, "requeue-after", result.RequeueAfter)
      // 如果删除失败，则需要在 ac 对象的事件中记录
			r.record.Event(ac, event.Warning(reasonCannotFinalizeWorkloads, err))
      // 在 ac status 字段中记录 Finalize 删除失败记录
			ac.SetConditions(v1alpha1.ReconcileError(errors.Wrap(err, errFinalizeWorkloads)))
   		// 退出当前主流程，并更新 ac status
			return reconcile.Result{}, errors.Wrap(r.UpdateStatus(ctx, ac), errUpdateAppConfigStatus)
		}
    // 此处直接返回，后续逻辑就不需要处理了
		return reconcile.Result{}, errors.Wrap(r.client.Update(ctx, ac), errUpdateAppConfigStatus)
	}

	// make sure this is the last functional defer function to be called
	defer func() {
		// Make sure if error occurs, reconcile will not happen too frequency
		if returnErr != nil {
			result.RequeueAfter = 0
		}
	}()

	// execute the posthooks at the end no matter what
	defer func() {
    // 更新 ObservedGeneration 字段，用于表示改版本已经被控制器处理过
		updateObservedGeneration(ac)
    // 执行后置任务
		for name, hook := range r.postHooks {
			exeResult, err := hook.Exec(ctx, ac, log)
			if err != nil {
				log.Debug("Failed to execute post-hooks", "hook name", name, "error", err, "requeue-after", result.RequeueAfter)
        // 设置后置任务执行事件
				r.record.Event(ac, event.Warning(reasonCannotExecutePosthooks, err))
        // 设置 CR 中 status 的后置任务执行结果
				ac.SetConditions(v1alpha1.ReconcileError(errors.Wrap(err, errExecutePosthooks)))	
				return
			}
		}
	}()

	// execute the prehooks
  // 执行前置任务，处理流程参考后置任务
	for name, hook := range r.preHooks {
		result, err := hook.Exec(ctx, ac, log)
		...
	}

  // 将 resource version 和 uuid 初始化到日志对象，后面每条日志将带上相关信息
	log = log.WithValues("uid", ac.GetUID(), "version", ac.GetResourceVersion())

  // 将 application 中 component 配置 渲染成 `内部 workload` 对象、以及依赖关系
	workloads, depStatus, err := r.components.Render(ctx, ac)
	
  ...

  // 增加 工作负载 提交附加options，主要用于校验该资源是否应该被创建，如果校验失败，则返回错误终止创建
	applyOpts := []apply.ApplyOption{apply.MustBeControllableBy(ac.GetUID()), applyOnceOnly(ac, r.applyOnceOnlyMode, log)}
  // 对前面渲染出来的工作负载提交创建，其中主要逻辑包含：
  // 1. 如果工作负载没有依赖其他负载或者依赖负载已经就绪，首先进行将 输入数据 渲染进工作负载，然后在进行工作负载的创建
  // 2. 创建 输出对象，从正在运行的工作负载中获取需要输出数据，将数据渲染进 输出对象，然后对 输出对象进行创建
  // 3. 如果 trait 没有依赖或者依赖trait已经就绪，首先将 输入数据 渲染到 trait 对象，对渲染之后对 trait 对象进行创建操作
  // 4. 最后进行 scope 创建
	if err := r.workloads.Apply(ctx, ac.Status.Workloads, workloads, applyOpts...); err != nil {
  ...
  }

	// Kubernetes garbage collection will (by default) reap workloads and traits
	// when the appconfig that controls them (in the controller reference sense)
	// is deleted. Here we cover the case in which a component or one of its
	// traits is removed from an extant appconfig.
  // 对所有工作负载执行垃圾收集操作
	for _, e := range r.gc.Eligible(ac.GetNamespace(), ac.Status.Workloads, workloads) {
		// https://github.com/golang/go/wiki/CommonMistakes#using-reference-to-loop-iterator-variable
		e := e

		log := log.WithValues("kind", e.GetKind(), "name", e.GetName())
		record := r.record.WithAnnotations("kind", e.GetKind(), "name", e.GetName())
    
    // 判断当前工作负载是否开始 ApplyOnceMode，如果未开启则继续向下执行
    // 如果开启该模式，且该工作负载副本数超过0，则不允许执行垃圾回收操作
    err := r.confirmDeleteOnApplyOnceMode(ctx, ac.GetNamespace(), &e)
		...

    // 执行删除操作
		if err := r.client.Delete(ctx, &e); resource.IgnoreNotFound(err) != nil {
		...
		}
	}

	// patch the final status on the client side, k8s sever can't merge them
	r.updateStatus(ctx, ac, acPatch, workloads)

	ac.Status.Dependency = v1alpha2.DependencyStatus{}
	var waitTime time.Duration
  // 如果还有依赖的工作负载没有就绪
	if len(depStatus.Unsatisfied) != 0 {
    // 设置等待时间，默认等待10秒之后，在进行对账
		waitTime = dependCheckWait
		ac.Status.Dependency = *depStatus
	}

	// the posthook function will do the final status update
	return reconcile.Result{RequeueAfter: waitTime}, nil
}
```

##### component render

```go
// A Workload produced by an OAM ApplicationConfiguration.
// Workload，内部转换对象，由 ApplicationConfiguration 对象转换而来
type Workload struct {
	// ComponentName that produced this workload.
  // component 名称
	ComponentName string

	// ComponentRevisionName of current component
  // component revision 名称
	ComponentRevisionName string

	// A Workload object.
  // Workload，这里的对象对应 CRD WorkloadDefinition
	Workload *unstructured.Unstructured

	// SkipApply indicates that the workload should not be applied
  // 该开关主要用于避免与 rollout 控制器冲突，在 rollout 阶段将不进行 apply 操作
	SkipApply bool

	// HasDep indicates whether this resource has dependencies and unready to be applied.
  // 主要用于标识当前资源是否有依赖关系以及是否 apply 就绪
	HasDep bool

	// Traits associated with this workload.
  // 用于存储当前 workload 绑定的 trait
	Traits []*Trait

	// RevisionEnabled means multiple workloads of same component will possibly be alive.
  // 当前 workload 是否开启 revision，如果开启，component 将会存在多个版本
	RevisionEnabled bool

	// Scopes associated with this workload.
  // 存储关联 scope 对象
	Scopes []unstructured.Unstructured

	// Record the DataOutputs of this workload, key is name of DataOutput.
  // 是否需要导出数据供外部使用，使用configmap进行存储
	DataOutputs map[string]v1alpha2.DataOutput

	// Record the DataInputs of this workload.
  // 是否依赖第三方提供数据，如果数据未就绪，该 workload 将不会被 apply 
	DataInputs []v1alpha2.DataInput
}
```

- Render

```go
func (r *components) Render(ctx context.Context, ac *v1alpha2.ApplicationConfiguration) ([]Workload, *v1alpha2.DependencyStatus, error) {
	workloads := make([]*Workload, 0, len(ac.Spec.Components))
	dag := newDAG()

	// we have special logics for application generated applicationConfiguration during rolling out phase
  // 新建 rolling component 集合
	rollingComponents := make(map[string]bool)
  // 初始化 needRolloutTemplate
	var needRolloutTemplate bool
  // 获取 ApplicationConfiguration annotation app.oam.dev/rollout-template，这个标记主要用于标记当前 ac 是否处于 rolling ，如果处于 rolling 则不应该对当前对象进行 apply 操作
	if _, isAppRolling := ac.GetAnnotations()[oam.AnnotationAppRollout]; isAppRolling {
		// we only care about the new components when there is rolling out
    // 检查 app.oam.dev/rolling-components 标记，如果该标记存在，value 中记录的是当前 ac 中所有正在 rolling 的 component 列表，用逗号进行分割
		if anc, exist := ac.GetAnnotations()[oam.AnnotationRollingComponent]; exist {
			// the rolling components annotation contains all the rolling components in the application
			for _, componentName := range strings.Split(anc, common.RollingComponentsSep) {
        // 将 app.oam.dev/rolling-components 标记的组件，添加到上面 rollingComponents 集合中
				rollingComponents[componentName] = true
			}
		}
		// we need to do a template roll out if it's not done yet
    // 如果当前 ac 状态不处于 rolling，则进行标记为需要生成 rolling 模版并执行 rolling， RollingTemplated 该状态主要用于说明当前 ac 正在进行 rolling 并且已经生成 rolling 模版
		needRolloutTemplate = ac.Status.RollingStatus != v1alpha2.RollingTemplated
	} else if ac.Status.RollingStatus == v1alpha2.RollingTemplated {
    // 如果当前 ac 已经 rolling 完毕，设置 ac 状态为 RollingComplete
		ac.Status.RollingStatus = v1alpha2.RollingComplete
	}

	for _, acc := range ac.Spec.Components {
    // 如果配置 revisionName，则需要从 revsionName 中提取真实的 componentName，一般 revisionName 类似 test-v1
		if acc.RevisionName != "" {
			acc.ComponentName = utils.ExtractComponentName(acc.RevisionName)
		}
    // 获取当前组件是否处于 rolling
		isComponentRolling := rollingComponents[acc.ComponentName]
    // 渲染 component 对象
		w, err := r.renderComponent(ctx, acc, ac, isControlledByApp(ac), isComponentRolling, needRolloutTemplate, dag)
		if err != nil {
			return nil, nil, err
		}
		workloads = append(workloads, w)
    // 如果 component 处于正在或者需要滚动发布的阶段
		if isComponentRolling && needRolloutTemplate {
      // 则将 ac 的滚动状态 设置成RollingTemplating
			ac.Status.RollingStatus = v1alpha2.RollingTemplating
		}
	}
  
  // 用于标记当前所有互相依赖的工作负载全部就绪
	workloadsAllClear := true
	ds := &v1alpha2.DependencyStatus{}
	res := make([]Workload, 0, len(ac.Spec.Components))
	for i, acc := range ac.Spec.Components {
    // 校验当前 component 与 ac 中存在依赖关系其他component 是否全部就绪，其中包括：
    // 1. component之间的 输入和输出对象是否都就绪
    // 2. trait之间的输入和输出对象是否都就绪
		unsatisfied, err := r.handleDependency(ctx, workloads[i], acc, dag, ac)
		if err != nil {
			return nil, nil, err
		}
    // ds 中记录没有满足的依赖关系的记录
		ds.Unsatisfied = append(ds.Unsatisfied, unsatisfied...)
		if workloads[i].HasDep {
			workloadsAllClear = false
		}
		res = append(res, *workloads[i])
	}
	// set the ac rollingStatus to be RollingTemplated if all workloads are going to be applied
  // 如果 ac 所有工作负载全部提交就绪，且 ac状态处于滚动中，则将ac状态修改为完成滚动
	if workloadsAllClear && ac.Status.RollingStatus == v1alpha2.RollingTemplating {
		ac.Status.RollingStatus = v1alpha2.RollingTemplated
	}

  // 最后返回渲染完成的 工作负载对象 和 DS对象
	return res, ds, nil
}
```

- renderComponent

```go
func (r *components) renderComponent(ctx context.Context, acc v1alpha2.ApplicationConfigurationComponent,
	ac *v1alpha2.ApplicationConfiguration, isControlledByApp, isComponentRolling, needRolloutTemplate bool, dag *dag) (*Workload, error) {
  // 如果配置revisionName 需要从中获取 componentName
	if acc.RevisionName != "" {
		acc.ComponentName = utils.ExtractComponentName(acc.RevisionName)
	}
	klog.InfoS("render a component", "component name", acc.ComponentName, "component revision", acc.RevisionName,
		"is generated by application", isControlledByApp, "is the component rolling", isComponentRolling,
		"is the appConfig a rollout template", needRolloutTemplate)

  // 获取 component 对象以及 revisionName，主要分两种情况：
  // 1. 如果 acc 中 revisionName 不为空，则查询对应的 appsv1.ControllerRevision，从 json 中解出 component 的对象
  // 2. 如果为空，则通过 componentName 直接查询对应 component 对象，并从 status 中读取 status.LatestRevision.Name
	c, componentRevisionName, err := util.GetComponent(ctx, r.client, acc, ac.GetNamespace())
	if err != nil {
		return nil, err
	}
  
  // 校验 component 中已经支持的 Parameters 和 acc 中配置的 Parameters，主要完成下面工作：
  // 1. 校验 acc 中是否配置未支持的 Parameters
  // 2. 校验 component 中声明为必填项的参数， acc中是否配置
  // 3. 返回 component 中配置的 Parameters 对应的路径，比如 spec.replicas 等等,该路径比较重要，后续将 Parameters 渲染到 workload 中时需要按照路径进行设置
	p, err := r.params.Resolve(c.Spec.Parameters, acc.ParameterValues)
	if err != nil {
		return nil, errors.Wrapf(err, errFmtResolveParams, acc.ComponentName)
	}

  // 将上一步获取的 Parameters 渲染到从 component 获取的 workload 对象中，生成完整的 component 对象 
	w, err := r.workload.Render(c.Spec.Workload.Raw, p...)
	if err != nil {
		return nil, errors.Wrapf(err, errFmtRenderWorkload, acc.ComponentName)
	}
	compInfoLabels := map[string]string{
		oam.LabelAppName:              ac.Name,
		oam.LabelAppComponent:         acc.ComponentName,
		oam.LabelAppComponentRevision: componentRevisionName,
		oam.LabelOAMResourceType:      oam.ResourceTypeWorkload,
	}
  // 添加标签，包括：ac名称、组件名称、组件版本、workload type
	util.AddLabels(w, compInfoLabels)

	compInfoAnnotations := map[string]string{
		oam.AnnotationAppGeneration: strconv.Itoa(int(ac.Generation)),
	}
  // 添加注释，用于标记当前 ac 配置文件版本号
	util.AddAnnotations(w, compInfoAnnotations)

	// pass through labels and annotation from app-config to workload
  // 透传 ac 中注释到 component
	util.PassLabelAndAnnotation(ac, w)
	// don't pass the following annotation as those are for appConfig only
  // 过滤 rolling 相关标签到 component
	util.RemoveAnnotations(w, []string{oam.AnnotationAppRollout, oam.AnnotationRollingComponent})
  // 获取当前 ac OwnerReference 信息
	ref := metav1.NewControllerRef(ac, v1alpha2.ApplicationConfigurationGroupVersionKind)
  // 设置 component OwnerReference，将 component 与 ac 的级联关系，方便后续做 gc
	w.SetOwnerReferences([]metav1.OwnerReference{*ref})
  // 设置 component namespace 信息
	w.SetNamespace(ac.GetNamespace())
  // 至此 component 全部渲染成功

  // 开始渲染 trait 对象 
	traits := make([]*Trait, 0, len(acc.Traits))
	traitDefs := make([]v1alpha2.TraitDefinition, 0, len(acc.Traits))
	compInfoLabels[oam.LabelOAMResourceType] = oam.ResourceTypeTrait

	for _, ct := range acc.Traits {
    // 渲染出 trait 、TraitDefinition 对象
		t, traitDef, err := r.renderTrait(ctx, ct, ac, acc.ComponentName, ref, dag)
		if err != nil {
			return nil, err
		}
    // 添加标签，包括：ac名称、组件名称、组件版本、workload type
		util.AddLabels(t, compInfoLabels)
    // 添加注释，用于标记当前 ac 配置文件版本号
		util.AddAnnotations(t, compInfoAnnotations)

		// pass through labels and annotation from app-config to trait
    // 透传 ac 中注释到 component
		util.PassLabelAndAnnotation(ac, t)
    // 过滤 rolling 相关标签到 component
		util.RemoveAnnotations(t, []string{oam.AnnotationAppRollout, oam.AnnotationRollingComponent})
		traits = append(traits, &Trait{Object: *t, Definition: *traitDef})
		traitDefs = append(traitDefs, *traitDef)
	}
  
  // 如果不受 application 对象管理
	if !isControlledByApp {
		// This is the legacy standalone appConfig approach
    // 获取当前正在运行中的工作负载
		existingWorkload, err := r.getExistingWorkload(ctx, ac, c, w)
		if err != nil {
			return nil, err
		}
    // 设置工作负载的名字，主要逻辑如下：
    // 1. 如果当前工作负载名称不为空，则直接退出
    // 2. 如果开启版本控制，但是 Status.LatestRevision 为空则退出
    // 3. 如果开启版本控制，且当前工作负载正在运行、版本没有发生变化，则保留当前工作负载的名字
    // 4. 如果开启版本控制，且工作负载名字跟 componentName 不一致，则使用 componentName 作为工作负载名字
    // 5. 如果没有开启版本控制，则一直用 componentName 作为 工作负载名字
		if err := setWorkloadInstanceName(traitDefs, w, c, existingWorkload); err != nil {
			return nil, err
		}
    // 如果受 application 对象管理
	} else {
		// we have a completely different approach on workload name for application generated appConfig
    // 获取当前 ac component 对应的版本号
		revision, err := utils.ExtractRevision(acc.RevisionName)
		if err != nil {
			return nil, err
		}
    // 设置工作负载名字，此处的规则为 componentName-revision，但是针对 kruise 中支持原地重启的工作负载使用 componentName 作为工作负载名字
		SetAppWorkloadInstanceName(acc.ComponentName, w, revision)
    // 针对正在或者需要进行滚动升级的工作负载需要特殊处理
		if isComponentRolling && needRolloutTemplate {
			// we have a special logic to emit the workload as a template so that the rollout
			// controller can take over.
      // 对于 kruise 中 CloneSet、StatefulSet、以及原生deployment 三种工作负载，在首次渲染工作负载配置时，需要将pause 字段设置为true，即上述三种工作负载将接手控制，进行二次渲染。
			if err := prepWorkloadInstanceForRollout(w); err != nil {
				return nil, err
			}
			klog.InfoS("Successfully rendered a workload instance for rollout", "workload", w.GetName())
		}
	}

	// create the ref after the workload name is set
	workloadRef := runtimev1alpha1.TypedReference{
		APIVersion: w.GetAPIVersion(),
		Kind:       w.GetKind(),
		Name:       w.GetName(),
	}
	//  We only patch a TypedReference object to the trait if it asks for it
  // 修改 trait 中 TypedReference 字段
	for i := range acc.Traits {
		traitDef := traitDefs[i]
		trait := traits[i]
		workloadRefPath := traitDef.Spec.WorkloadRefPath
		if len(workloadRefPath) != 0 {
			if err := fieldpath.Pave(trait.Object.UnstructuredContent()).SetValue(workloadRefPath, workloadRef); err != nil {
				return nil, errors.Wrapf(err, errFmtSetWorkloadRef, trait.Object.GetName(), w.GetName())
			}
		}
	}
  
  // 渲染 scope 对象
	scopes := make([]unstructured.Unstructured, 0, len(acc.Scopes))
	for _, cs := range acc.Scopes {
		scopeObject, err := r.renderScope(ctx, cs, ac.GetNamespace())
		if err != nil {
			return nil, err
		}

		scopes = append(scopes, *scopeObject)
	}

	addDataOutputsToDAG(dag, acc.DataOutputs, w)
	// To avoid conflict with rollout controller, we will not render the workload until the rollout phase is over
	// indicated by the AnnotationAppRollout annotation disappear
	return &Workload{ComponentName: acc.ComponentName, ComponentRevisionName: componentRevisionName,
		SkipApply: isComponentRolling && !needRolloutTemplate,
		Workload:  w, Traits: traits, RevisionEnabled: isRevisionEnabled(traitDefs), Scopes: scopes}, nil
}
```

##### trait render

```go
func (r *components) renderTrait(ctx context.Context, ct v1alpha2.ComponentTrait, ac *v1alpha2.ApplicationConfiguration,
	componentName string, ref *metav1.OwnerReference, dag *dag) (*unstructured.Unstructured, *v1alpha2.TraitDefinition, error) {
  // 将 ac 中 trait 相关配置，渲染成 trait 对象
	t, err := r.trait.Render(ct.Trait.Raw)
	if err != nil {
		return nil, nil, errors.Wrapf(err, errFmtRenderTrait, componentName)
	}
  
  // 获取 ac 中 trait 配置到指定的 traitDefinition 对象
	traitDef, err := util.FetchTraitDefinition(ctx, r.client, r.dm, t)
	if err != nil {
		if !apierrors.IsNotFound(err) {
			return nil, nil, errors.Wrapf(err, errFmtGetTraitDefinition, t.GetAPIVersion(), t.GetKind(), t.GetName())
		}
    // 如果没有获取对应的 traitDefinition 对象，则会伪造一个 dummy 对象，后续流程会处理这个类型的对象
		traitDef = util.GetDummyTraitDefinition(t)
	}

  // 获取 trait 对象，主要逻辑包括：
  // 1. 如果 traitName 不为空且 trait 不由 application 管理，则直接返回当前 traitName
  // 2. 其他情况下，将生成 traitName，格式按照 %componentName%-trait-%hash% ，这里面的hash 按照 component Trait 配置生成，参考 deployment 中 hash 的生成
	traitName := getTraitName(ac, componentName, &ct, t, traitDef)
  // 对 trait 对象进行参数设置，包括：traitName、namespaces、OwnerReferences
	setTraitProperties(t, traitName, ac.GetNamespace(), ref)

	addDataOutputsToDAG(dag, ct.DataOutputs, t)

	return t, traitDef, nil
}
```

#### Application(app)

##### CRD

```go
type ApplicationTrait struct {
  // 用于创建 trait 对象时的名称
	Name string `json:"name"`

  // 用于渲染 TraitDefinition 中 parameter 部分的参数
	Properties runtime.RawExtension `json:"properties"`
}

// ApplicationComponent describe the component of application
type ApplicationComponent struct {
  // component 名称，用于创建 component 对象时使用
	Name         string `json:"name"`
  // 用于指定 WorkloadDefinition 名称
	WorkloadType string `json:"type"`
  // 用于渲染 WorkloadDefinition  中 parameter 部分的参数
	Settings runtime.RawExtension `json:"settings"`
  // 用于定义哪些 trait 需要挂载到 component
	Traits []ApplicationTrait `json:"traits,omitempty"`

	Scopes map[string]string `json:"scopes,omitempty"`
}

// ApplicationSpec is the spec of Application
type ApplicationSpec struct {
  // 用于定义多个组件，相当于前面 ac 中的 component 对象
	Components []ApplicationComponent `json:"components"`

	RolloutPlan *v1alpha1.RolloutPlan `json:"rolloutPlan,omitempty"`
}

type Application struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ApplicationSpec `json:"spec,omitempty"`
	Status AppStatus       `json:"status,omitempty"`
}
```

##### Reconcile

```go
// Appfile describes application
// 该对象用于描述 application 对象
type Appfile struct {
	Name      string
	Workloads []*Workload
}
```



```go
func (r *Reconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	ctx := context.Background()
	applog := r.Log.WithValues("application", req.NamespacedName)
	app := new(v1alpha2.Application)
  // 获取入队的 application 对象
	if err := r.Get(ctx, client.ObjectKey{
		Name:      req.Name,
		Namespace: req.Namespace,
	}, app); err != nil {
    // 如果还未创建，则退出
		if kerrors.IsNotFound(err) {
			err = nil
		}
		return ctrl.Result{}, err
	}

	// TODO: check finalizer
  // 如果正在被删除，则退出
	if app.DeletionTimestamp != nil {
		return ctrl.Result{}, nil
	}

	applog.Info("Start Rendering")

  // 设置 app 状态为 渲染中
	app.Status.Phase = v1alpha2.ApplicationRendering
	handler := &appHandler{r, app, applog}

	applog.Info("parse template")
	// parse template
	appParser := appfile.NewApplicationParser(r.Client, r.dm)

	ctx = oamutil.SetNnamespaceInCtx(ctx, app.Namespace)

  // 将 app 转换成 上面 Appfile 对象，最核心的转换是将 app 中 component、trait、scope 对象转换成 内部 Workload 对象
	appfile, err := appParser.GenerateAppFile(ctx, app.Name, app)
	if err != nil {
		applog.Error(err, "[Handle Parse]")
		app.Status.SetConditions(errorCondition("Parsed", err))
		return handler.handleErr(err)
	}

  // 设置 app 状态 为 完成解析
	app.Status.SetConditions(readyCondition("Parsed"))

	applog.Info("build template")

  // 从 appfile 转换成 ac 和 component，主要转换逻辑：
  // 1. 从 ac 中拿到所有 workload，然后从 WorkloadDefinition 和 TraitDefinition 中获取对应对象
  // 2. 从 Definition 中拿到 cue 定义的模版部分，将 parameter 部分数据渲染到 outputField中完成数据渲染，
  // 3. 将 workload 渲染出来的数据构建成 Component 对象，将 trait、scope 渲染出来的数据构建 acCompoent
	ac, comps, err := appParser.GenerateApplicationConfiguration(appfile, app.Namespace)
  
	...
  
  // 创建上面渲染出来的 ac 和 compoent 组件
	if err := handler.apply(ctx, ac, comps); err != nil {
	}

	// 检查 app 健康状态，如果失败则退出，健康检查主要检查所有对象中是否有 isHealth 字段，如果存在则说明健康。
	appCompStatus, healthy, err := handler.statusAggregate(appfile)
	if err != nil {
	}
	if !healthy {
		// 如果不健康则 10 秒之后在重试
	}
	app.Status.Services = appCompStatus
	app.Status.SetConditions(readyCondition("HealthCheck"))
	app.Status.Phase = v1alpha2.ApplicationRunning
	// Gather status of components
  // 生成所有组件的状态，包括 gvk、name、uuid
	var refComps []v1alpha1.TypedReference
	for _, comp := range comps {
		refComps = append(refComps, v1alpha1.TypedReference{
			APIVersion: comp.APIVersion,
			Kind:       comp.Kind,
			Name:       comp.Name,
			UID:        app.UID,
		})
	}
	app.Status.Components = refComps
	return ctrl.Result{}, r.UpdateStatus(ctx, app)
}
```

#### ApplicationDeployment(ad)

##### CRD

```go
type AppRollout struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   AppRolloutSpec   `json:"spec,omitempty"`
	Status AppRolloutStatus `json:"status,omitempty"`
}

// AppRolloutSpec defines how to describe an upgrade between different apps
type AppRolloutSpec struct {
  // TargetAppRevisionName，用于配置需要升级到哪个版本的 ac，这里配置 ac 名字即可，必填字段
	TargetAppRevisionName string `json:"targetApplicationName"`

  // SourceApplicationName，对应上一个字段，此字段用于配置哪个版本的 ac 需要升级，该字段只有在首次发布时可以为空
	SourceApplicationName string `json:"sourceApplicationName,omitempty"`

  // ComponentList，对于 ac 中需要更新的 component 列表
  // 目前只支持单 component 的 ac 进行更新，后续会支持多 component
	ComponentList []string `json:"componentList,omitempty"`

	// RolloutPlan，用于定义滚动升级具体执行计划
	RolloutPlan v1alpha1.RolloutPlan `json:"rolloutPlan"`

	// RevertOnDelete revert the rollout when the rollout CR is deleted
	// It will remove the target app from the kubernetes if it's set to true
	// +optional
	RevertOnDelete *bool `json:"revertOnDelete,omitempty"`
}

// RolloutPlan fines the details of the rollout plan
type RolloutPlan struct {

	// RolloutStrategy ，用于定义滚动计划的策略：IncreaseFirst（新增加）、DecreaseFirst（先减少）
  // 这个配置类似于 deployment RollingUpdateStrategy ，关于 max unavailable, max surge 的定义
	// +optional
	RolloutStrategy *RolloutStrategyType `json:"rolloutStrategy,omitempty"`

	// TargetSize，用于定义需要执行滚动升级的总副本数
	// +optional
	TargetSize *int32 `json:"targetSize,omitempty"`

	// NumBatches，用于定义分几批进行更新
	// +optional
	NumBatches *int32 `json:"numBatches,omitempty"`

  // 每次分批的确切信息，里面包含当前批次：副本数、pod 列表、最大不可用 pod 数、滚动间隔、分批滚动 webhook、监控指标
	RolloutBatches []RolloutBatch `json:"rolloutBatches,omitempty"`

	// 该字段应该是类似于 sts 中的 Partition，进行分区更新， 所有序号大于等于该分区序号的 Pod 都会被更新。 
  // 所有序号小于该分区序号的 Pod 都不会被更新，并且，即使他们被删除也会依据之前的版本进行重建
	BatchPartition *int32 `json:"batchPartition,omitempty"`

	// 滚动暂停，类似deployment 中 paused 字段，当前分批执行滚动之后，下一批才能暂停。
	// +optional
	Paused bool `json:"paused,omitempty"`

	// RolloutWebhooks，每次分批更新都会出发一次外部接口调用
	// +optional
	RolloutWebhooks []RolloutWebhook `json:"rolloutWebhooks,omitempty"`

	// CanaryMetric，用于定义 判定当前更新是否有效，以及是否进行下批更新的标准，类似于 argocd 里面的 metrics指标，比如 http 502 请求qps 低于某个阈值判定更新有效，否则将会进行回滚
	// before complete the process
	// +optional
	CanaryMetric []CanaryMetric `json:"canaryMetric,omitempty"`
}
```

##### Reconcile

```go
// applicationdeploymentController
// Reconcile is the main logic of applicationdeployment controller
func (r *Reconciler) Reconcile(req ctrl.Request) (res reconcile.Result, retErr error) {
	// 这里省略初始化 ad 对象，和获取当前入队 ad 对象
  ...

	// TODO: check if the target/source has changed
  // 预期：ad 对象被标记为待删除时，拦截器检查入队 ad 对象 目标和源 ac 对象是否发生变化
	r.handleFinalizer(&appRollout)

	ctx = oamutil.SetNamespaceInCtx(ctx, appRollout.Namespace)

	// Get the target application
  // 获取目标、源 ac 对象
	var targetApp oamv1alpha2.ApplicationConfiguration
	sourceApp := &oamv1alpha2.ApplicationConfiguration{}
  // 获取目标 ac 当前版本名称
	targetAppName := appRollout.Spec.TargetAppRevisionName
	if err := r.Get(ctx, ktypes.NamespacedName{Namespace: req.Namespace, Name: targetAppName},
		&targetApp); err != nil {
		...
	}

	// Get the source application
  // 获取源 ac 对象当前版本名称
	sourceAppName := appRollout.Spec.SourceApplicationName
  // 如果 ad 中没有配置源 ac 对象版本名称，则说明当前发布中的应用时首次发布
	if sourceAppName == "" {
		klog.Info("source app fields not filled, we assume it is deployed for the first time")
		sourceApp = nil
    // 如果源 ac 对象版本名称存在，则从 k8s 中获取当前 ac 对象
	} else if err := r.Get(ctx, ktypes.NamespacedName{Namespace: req.Namespace, Name: sourceAppName}, sourceApp); err != nil {
			。。。
	}

  // 将目标、源 ac 对象中的 component 组件渲染成内部 `Workload` 对象，主要逻辑如下：
  // 1. 如果 ad 对象中 Spec.ComponentList 中没有定义 component，则从所有 ac 对象中获取一个主要的 component 来进行滚动升级，前面也说过，目前 ad 只支持对单 component 的 ac 对象进行滚动升级，后续会支持多 component
  // 2. 如果 ad 对象中 Spec.ComponentList 以及声明，默认从列表中获取第一个 component 进行滚动升级
  // 3. 以上以及找到目标、源 ac 中需要滚动更新的 componentName，这里名称应该是一致的
  // 4. 按照 componentName 获取最底层的工作负载，这里的工作负载指 WorkloadDefin 中定义的。这里获取方法如下：
  // 		a. 从 ac 对象中获取 componentName 对应的 acc 对象，acc 对象指的 ApplicationConfigurationComponent
  // 		b. 从上面 acc 对象中获取版本名称，revisionname，并从中提取出版本号
  // 		c. 如果上面 acc 对象 revisionName 不为空，则用 revisionName 获取当前 k8s 中实例化之后的 component 资源，如果没有配置revisionName 则用 componentName 获取
  // 		d. 从上面获取的 component 对象中拿到真实工作负载的对象，并且转换为 unstructured.Unstructured 对象
  // 		e. 这里需要处理几种特殊工作负载：openkruise 中的 cloneset/statefulset，component 的名称和最终落到 k8s 中对象名称是不一致的，最终的名字一直是componentName，如果使用revisionName 话会影响上述两个工作负载的 in-place 即原地升级功能能，所以这里需要调用 SetAppWorkloadInstanceName 来做一次适配
  // 		f. 通过前面获取的 namespace、gvk、name 等信息从 k8s 总获取正在运行中的工作负载对象，即 deployment、sts等等工作负载
  // 5. 按照上面的方法获取目标、源 ac 中真正允许在 k8s 中的工作负载对象
	targetWorkload, sourceWorkload, err := r.extractWorkloads(ctx, appRollout.Spec.ComponentList, &targetApp, sourceApp)
	if err != nil {
		...
		return ctrl.Result{RequeueAfter: 5 * time.Second}, client.IgnoreNotFound(err)
	}
	klog.InfoS("get the target workload we need to work on", "targetWorkload", klog.KObj(targetWorkload))

	if sourceWorkload != nil {
		klog.InfoS("get the source workload we need to work on", "sourceWorkload", klog.KObj(sourceWorkload))
	}
  // 上述已经完成目标、源真正需要操作的工作负载对象，下面就进入滚动升级的核心控制器 rolloutPlanController

	// reconcile the rollout part of the spec given the target and source workload
	rolloutPlanController := rollout.NewRolloutPlanController(r, &appRollout, r.record,
		&appRollout.Spec.RolloutPlan, &appRollout.Status.RolloutStatus, targetWorkload, sourceWorkload)
	result, rolloutStatus := rolloutPlanController.Reconcile(ctx)
	// make sure that the new status is copied back
	appRollout.Status.RolloutStatus = *rolloutStatus
	if rolloutStatus.RollingState == v1alpha1.RolloutSucceedState {
		// remove the rollout annotation so that the target appConfig controller can take over the rest of the work
		oamutil.RemoveAnnotations(&targetApp, []string{oam.AnnotationAppRollout})
		if err := r.Update(ctx, &targetApp); err != nil {
			klog.ErrorS(err, "cannot remove the rollout annotation", "target application",
				klog.KRef(req.Namespace, targetAppName))
			return ctrl.Result{}, err
		}
	}
	// update the appRollout status
	return result, r.updateStatus(ctx, &appRollout)
}
```

```go
// rolloutPlanController
// Reconcile reconciles a rollout plan
func (r *Controller) Reconcile(ctx context.Context) (res reconcile.Result, status *v1alpha1.RolloutStatus) {
	klog.InfoS("Reconcile the rollout plan", "rollout Spec", r.rolloutSpec,
		"target workload", klog.KObj(r.targetWorkload))
	if r.sourceWorkload != nil {
		klog.InfoS("We will do rolling upgrades", "source workload", klog.KObj(r.sourceWorkload))
	}
	klog.InfoS("rollout status", "rollout state", r.rolloutStatus.RollingState, "batch rolling state",
		r.rolloutStatus.BatchRollingState, "current batch", r.rolloutStatus.CurrentBatch, "upgraded Replicas",
		r.rolloutStatus.UpgradedReplicas)

	defer func() {
		klog.InfoS("Finished reconciling rollout plan", "rollout state", status.RollingState,
			"batch rolling state", status.BatchRollingState, "current batch", status.CurrentBatch,
			"upgraded Replicas", status.UpgradedReplicas, "reconcile result ", res)
	}()
	status = r.rolloutStatus

	defer func() {
		if status.RollingState == v1alpha1.RolloutFailedState ||
			status.RollingState == v1alpha1.RolloutSucceedState {
			// no need to requeue if we reach the terminal states
			res = reconcile.Result{}
		} else {
			res = reconcile.Result{
				RequeueAfter: rolloutReconcileRequeueTime,
			}
		}
	}()

	workloadController, err := r.GetWorkloadController()
	if err != nil {
		r.rolloutStatus.RolloutFailed(err.Error())
		r.recorder.Event(r.parentController, event.Warning("Unsupported workload", err))
		return
	}

	switch r.rolloutStatus.RollingState {
	case v1alpha1.VerifyingState:
		r.rolloutStatus = workloadController.Verify(ctx)

	case v1alpha1.InitializingState:
		if err := r.initializeRollout(ctx); err == nil {
			r.rolloutStatus = workloadController.Initialize(ctx)
		}

	case v1alpha1.RollingInBatchesState:
		r.reconcileBatchInRolling(ctx, workloadController)

	case v1alpha1.FinalisingState:
		r.rolloutStatus = workloadController.Finalize(ctx)
		// if we are still going to finalize it
		if r.rolloutStatus.RollingState == v1alpha1.FinalisingState {
			r.finalizeRollout(ctx)
		}

	case v1alpha1.RolloutSucceedState:
		// Nothing to do

	case v1alpha1.RolloutFailedState:
		// Nothing to do

	default:
		panic(fmt.Sprintf("illegal rollout status %+v", r.rolloutStatus))
	}

	return res, r.rolloutStatus
}
```



#### DAG