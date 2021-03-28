# GarbageCollection（gc）

## 描述

我们在对一组 Kubernetes 资源抽象成一个对象时，比如包含 deployment、service、ingress、hpa、pdb等资源，在新版中版本不需要 ingress 配置，我们在顶层对象中移除 ingress 配置，此时 Kubernetes 集群中 ingress 如果被销毁就成问题。本文主要就介绍在 `kubevela` 中，顶层对象 `ApplicationConfiguration` 如果处理 下属 trait 被移除配置后，如何进行 gc 资源。

![](/Users/zhuhuijun/go/src/github.com/ghostbaby/kubevela-cookbook/image/GarbageCollection.jpg)

## 筛选

```go
func eligible(namespace string, ws []v1alpha2.WorkloadStatus, w []Workload) []unstructured.Unstructured {
	applied := make(map[v1alpha1.TypedReference]bool)
  // 这里的 workload 切片为期望对象，即当前生效的对象列表
	for _, wl := range w {
		r := v1alpha1.TypedReference{
			APIVersion: wl.Workload.GetAPIVersion(),
			Kind:       wl.Workload.GetKind(),
			Name:       wl.Workload.GetName(),
		}
    // 将当前 workload 加入到 applied 列表
		applied[r] = true
		for _, t := range wl.Traits {
			r := v1alpha1.TypedReference{
				APIVersion: t.Object.GetAPIVersion(),
				Kind:       t.Object.GetKind(),
				Name:       t.Object.GetName(),
			}
      // 将当前 ac 中包含的 trait 加入到 applied 列表
			applied[r] = true
		}
	}
  // 初始化需要被 gc 到 Unstructured 列表
	eligible := make([]unstructured.Unstructured, 0)
  // 从 ac WorkloadStatus 列表中获取历史信息
	for _, s := range ws {
		// 如果 WorkloadStatus 中的对象不存在于最新生效对象列表 applied 中，同时 workload 没有开启 revision 功能，则回收该对象
		if !applied[s.Reference] && !IsRevisionWorkload(s, w) {
			w := &unstructured.Unstructured{}
			w.SetAPIVersion(s.Reference.APIVersion)
			w.SetKind(s.Reference.Kind)
			w.SetNamespace(namespace)
			w.SetName(s.Reference.Name)
      // 添加当前对象到待回收列表
			eligible = append(eligible, *w)
		}

		for _, ts := range s.Traits {
      // 如果当前对象不存在于 applied 中，则直接回收该对象
			if !applied[ts.Reference] {
				t := &unstructured.Unstructured{}
				t.SetAPIVersion(ts.Reference.APIVersion)
				t.SetKind(ts.Reference.Kind)
				t.SetNamespace(namespace)
				t.SetName(ts.Reference.Name)
				eligible = append(eligible, *t)
			}
		}
	}

	return eligible
}
```

## 核心逻辑

```go
// 筛选当前所有 workload 对象是否符合回收的标准
for _, e := range r.gc.Eligible(ac.GetNamespace(), ac.Status.Workloads, workloads) {
		// https://github.com/golang/go/wiki/CommonMistakes#using-reference-to-loop-iterator-variable
		e := e

		log := log.WithValues("kind", e.GetKind(), "name", e.GetName())
		record := r.record.WithAnnotations("kind", e.GetKind(), "name", e.GetName())

  	// 检查当前对象状态是否符合删除标准，以下情况不允许进行删除操作：
  	// 1. ApplyOnceOnlyOff 该开关关闭的对象
  	// 2. 如果当前对象副本数不为0，则说明当前对象有可能正在进行滚动升级
		err := r.confirmDeleteOnApplyOnceMode(ctx, ac.GetNamespace(), &e)
		if err != nil {
			log.Debug("confirm component can't be garbage collected", "error", err)
			record.Event(ac, event.Warning(reasonCannotGGComponents, err))
			ac.SetConditions(v1alpha1.ReconcileError(errors.Wrap(err, errGCComponent)))
			return reconcile.Result{}
		}
  	// 从 Kubernetes 中删除当前对象
		if err := r.client.Delete(ctx, &e); resource.IgnoreNotFound(err) != nil {
			log.Debug("Cannot garbage collect component", "error", err)
			record.Event(ac, event.Warning(reasonCannotGGComponents, err))
			ac.SetConditions(v1alpha1.ReconcileError(errors.Wrap(err, errGCComponent)))
			return reconcile.Result{}
		}
  	// 完成当前对象的回收
		log.Debug("Garbage collected resource")
  	// ac 对象写入回收成功的event
		record.Event(ac, event.Normal(reasonGGComponent, "Successfully garbage collected component"))
	}
```

