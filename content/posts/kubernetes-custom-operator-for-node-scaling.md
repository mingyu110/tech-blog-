---
title: "Kubernetes自定义控制器开发指南：从一个真实业务场景出发"
date: 2025-07-18
draft: false
tags: ["Kubernetes", "Operator", "Go", "Controller", "私有云", "自动化运维", "CRD", "自建K8s"]
categories: ["云原生", "Kubernetes"]
description: "本文档从一个私有云环境下的常见运维痛点——自动化节点缩容——出发，详细阐述了如何从零开始构建一个生产级的 Kubernetes 自定义控制器（Operator）。"
toc: true
---

## 1. 背景：在自建 Kubernetes 集群中实现自动化节点缩容

在公司当前使用的私有云环境中，Kubernetes 集群完全自建，不依赖任何云服务商的托管服务（如 AWS EKS、GKE 或 AKS）。这导致无法直接利用托管 Kubernetes 集群提供的自动扩缩容、节点管理等功能。然而，出于成本优化和运维稳定性需求，经常需要对 Kubernetes 集群中的指定节点进行缩容，例如优先移除低利用率或特定标签的节点。

参考 AWS 技术博客《Best Practices for Quickly Scaling Down Specified Node Instances in an Amazon EKS Cluster》，我们开发了一个自定义 Operator 和 CRD（Custom Resource Definition），实现对指定节点的自动化缩容功能。

本文档总结了开发过程、技术实现细节和最佳实践，旨在为类似自建环境中的开发者提供参考。

---

## 2. 核心原理与源码解析

要编写高质量的控制器，必须理解其底层的核心组件。这些组件大多位于 `k8s.io/client-go` 包中。

### 2.1 控制循环 (Reconcile Loop)

控制器的灵魂在于**调谐循环 (Reconcile Loop)**。其逻辑非常纯粹：
> 获取一个对象的 `namespace/name`，查询该对象的期望状态，获取集群中与之关联的实际状态，然后调整实际状态以匹配期望状态。

这个循环必须是幂等的，即无论执行多少次，只要期望状态不变，最终结果都应相同。

### 2.2 Informer：高效的资源监控

如果控制器频繁地向 API Server 发送 `GET` 和 `LIST` 请求来获取资源状态，会给 API Server 带来巨大压力。`Informer` 机制解决了这个问题。

- **工作机制**：`Informer` 在本地维护了一个特定资源（如 Pods、Nodes）的**内存缓存 (Cache)**。它通过与 API Server 建立一个长连接 (`Watch`) 来实时接收资源变更事件（`Added`, `Updated`, `Deleted`），并相应地更新本地缓存和触发事件回调。
- **源码关联**：在 `client-go/tools/cache` 目录下，`SharedInformer` 是最常用的实现。它内部使用 `Reflector` 来 `ListAndWatch` 资源，并将事件和对象存入 `DeltaFIFO` 队列，最终由 `Controller` 组件消费并更新 `Indexer`（一个带索引功能的本地缓存）。

使用 Informer 可以极大降低 API Server 的负载，并让控制器的事件响应更及时、高效。

### 2.3 Workqueue：可靠的事件处理

当 Informer 触发事件时，我们不能在 Informer 的回调函数中直接执行耗时的调谐逻辑，这会阻塞后续事件的处理。`Workqueue` (工作队列) 用于解耦“事件检测”和“逻辑处理”。

- **工作机制**：Informer 的事件处理器只做一件事：将发生变更的对象的 `namespace/name`（作为 key）推入一个 `Workqueue`。在另一端，控制器的工作线程（Worker）从队列中取出 key，执行完整的调谐逻辑。
- **核心优势**：
    1.  **去重**：如果在处理一个对象时，该对象又发生了多次变更，Workqueue 会自动将这些变更合并为一个 key，避免无效处理。
    2.  **速率限制**：如果处理某个 key 时发生错误，控制器会将其重新放回队列。`RateLimitingQueue` 实现（`client-go/util/workqueue`）提供了指数退避的重试机制，避免因某个故障对象导致无休止的“热循环 (Hot Loop)”。

### 2.4 `controller-runtime`：高级抽象

直接使用 `client-go` 的 Informer 和 Workqueue 编写控制器需要处理大量样板代码。`sigs.k8s.io/controller-runtime` 项目提供了更高层次的抽象，极大地简化了开发工作。

- **Manager (`mgr`)**：管理全局配置，如 `kubeconfig`、`scheme`（资源类型注册表），并负责启动所有控制器和 Webhook。
- **Controller**：封装了 Informer 和 Workqueue 的复杂性。你只需指定要“观察 (Watch)”的资源类型，它会自动处理事件并调用你的 Reconciler。
- **Reconciler**：这是开发者唯一需要实现的核心接口。它的 `Reconcile` 方法接收一个 `Request`（包含对象的 `namespace/name`），开发者只需在此方法内实现完整的调谐逻辑。

`controller-runtime` 让开发者能真正专注于业务逻辑，而不是底层机制的实现细节。

---

## 3. 实践：构建生产级节点缩容控制器

我们将构建一个 `NodeScaler` 控制器。其功能是：当一个节点被打上特定标签（如 `scaler.example.com/strategy: scale-down`）时，控制器将安全地排空 (drain) 该节点上的所有 Pod，然后调用云厂商 API 将其从集群中移除。

### 3.1 环境准备与项目脚手架

我们强烈推荐使用 `Kubebuilder` 来初始化项目，它能自动生成所有必需的样板代码。

'''bash
# 安装 Kubebuilder
# ...

# 初始化项目
mkdir node-scaler-operator && cd node-scaler-operator
kubebuilder init --domain example.com --repo example.com/node-scaler-operator

# 创建 API (CRD 和 Controller)
kubebuilder create api --group scaler --version v1 --kind NodeScaler
'''

### 3.2 API 定义 (CRD)

我们的 `NodeScaler` 资源不需要复杂的 `spec`，因为它是一个触发器。但我们会用 `status` 字段来记录其工作状态。

`api/v1/nodescaler_types.go`:
'''go
// NodeScalerSpec defines the desired state of NodeScaler
type NodeScalerSpec struct {
    // NodeName is the name of the node to be scaled down.
    // +kubebuilder:validation:Required
    NodeName string `json:"nodeName"`
}

// NodeScalerStatus defines the observed state of NodeScaler
type NodeScalerStatus struct {
    // Phase indicates the current state of the scaling process.
    // e.g., "Pending", "Cordoning", "Draining", "Terminating", "Succeeded", "Failed"
    Phase string `json:"phase,omitempty"`
    // Message provides more details about the current phase.
    Message string `json:"message,omitempty"`
}
'''
修改后，运行 `make manifests` 来更新 CRD 文件。

### 3.3 控制器实现

我们将重写 `controllers/nodescaler_controller.go`。

**关键重构点**：
1.  **不使用 `kubectl`**：所有操作都通过 `client-go` API 完成。
2.  **精细的错误处理**：对于可重试的错误，返回 `ctrl.Result{Requeue: true}` 或 `ctrl.Result{RequeueAfter: ...}`。
3.  **状态更新**：在关键步骤后，及时更新 `NodeScaler` 实例的 `status` 字段。
4.  **云平台解耦**：终止节点的逻辑将被封装在一个接口中，方便替换为不同的云提供商实现。

`controllers/nodescaler_controller.go`:
'''go
package controllers

import (
	"context"
	"fmt"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/drain"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	scalerv1 "example.com/node-scaler-operator/api/v1"
)

// NodeScalerReconciler reconciles a NodeScaler object
type NodeScalerReconciler struct {
	client.Client
	Scheme    *runtime.Scheme
	Clientset *kubernetes.Clientset // Use clientset for drain helper
}

// CloudProvider is an interface for cloud-specific operations
type CloudProvider interface {
	TerminateNode(ctx context.Context, node *corev1.Node) error
}

// AwsCloudProvider is a sample implementation for AWS
type AwsCloudProvider struct{}

func (p *AwsCloudProvider) TerminateNode(ctx context.Context, node *corev1.Node) error {
	// IMPORTANT: Production code should use the AWS SDK for Go.
	// This is a placeholder for demonstration.
	// 1. Get instance ID from node.Spec.ProviderID
	// 2. Call EC2 TerminateInstances API
	log.FromContext(ctx).Info("Simulating node termination", "node", node.Name, "providerID", node.Spec.ProviderID)
	// Example: cmd := exec.Command("aws", "ec2", "terminate-instances", "--instance-ids", instanceID)
	// return cmd.Run()
	return nil
}

//+kubebuilder:rbac:groups=scaler.example.com,resources=nodescalers,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=scaler.example.com,resources=nodescalers/status,verbs=get;update;patch
//+kubebuilder:rbac:groups="",resources=nodes,verbs=get;list;watch;update;patch
//+kubebuilder:rbac:groups="",resources=pods,verbs=list;watch
//+kubebuilder:rbac:groups="",resources=pods/eviction,verbs=create

func (r *NodeScalerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// 1. Fetch the NodeScaler instance
	scaler := &scalerv1.NodeScaler{}
	if err := r.Get(ctx, req.NamespacedName, scaler); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// If phase is already Succeeded or Failed, do nothing
	if scaler.Status.Phase == "Succeeded" || scaler.Status.Phase == "Failed" {
		return ctrl.Result{}, nil
	}

	// 2. Fetch the target Node
	node := &corev1.Node{}
	if err := r.Get(ctx, client.ObjectKey{Name: scaler.Spec.NodeName}, node); err != nil {
		if errors.IsNotFound(err) {
			logger.Error(err, "Target node not found")
			scaler.Status.Phase = "Failed"
			scaler.Status.Message = "Node not found."
			_ = r.Status().Update(ctx, scaler)
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}

	// 3. Cordon the node
	if !node.Spec.Unschedulable {
		logger.Info("Cordoning node...")
		scaler.Status.Phase = "Cordoning"
		scaler.Status.Message = "Marking node as unschedulable."
		if err := r.Status().Update(ctx, scaler); err != nil {
			return ctrl.Result{}, err
		}

		node.Spec.Unschedulable = true
		if err := r.Update(ctx, node); err != nil {
			logger.Error(err, "Failed to cordon node")
			return ctrl.Result{RequeueAfter: 10 * time.Second}, err
		}
		return ctrl.Result{Requeue: true}, nil
	}

	// 4. Drain the node
	if scaler.Status.Phase != "Draining" && scaler.Status.Phase != "Terminating" {
		logger.Info("Draining node...")
		scaler.Status.Phase = "Draining"
		scaler.Status.Message = "Evicting all pods from the node."
		if err := r.Status().Update(ctx, scaler); err != nil {
			return ctrl.Result{}, err
		}

		// Use the official drain helper
		drainer := &drain.Helper{
			Ctx:                 ctx,
			Client:              r.Clientset,
			Force:               true,
			IgnoreAllDaemonSets: true,
			DeleteEmptyDirData:  true,
			GracePeriodSeconds:  60,
			Out:                 logger.GetSink(),
			ErrOut:              logger.GetSink(),
		}

		if err := drain.RunNodeDrain(drainer, node.Name); err != nil {
			logger.Error(err, "Failed to drain node")
			scaler.Status.Phase = "Failed"
			scaler.Status.Message = fmt.Sprintf("Drain failed: %v", err)
			_ = r.Status().Update(ctx, scaler)
			return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
		}
		return ctrl.Result{Requeue: true}, nil
	}

	// 5. Terminate the node from cloud provider
	logger.Info("Terminating node from cloud provider...")
	scaler.Status.Phase = "Terminating"
	scaler.Status.Message = "Calling cloud provider to terminate the instance."
	if err := r.Status().Update(ctx, scaler); err != nil {
		return ctrl.Result{}, err
	}

	cloudProvider := &AwsCloudProvider{} // Replace with your provider
	if err := cloudProvider.TerminateNode(ctx, node); err != nil {
		logger.Error(err, "Failed to terminate node from cloud provider")
		scaler.Status.Phase = "Failed"
		scaler.Status.Message = fmt.Sprintf("Termination failed: %v", err)
		_ = r.Status().Update(ctx, scaler)
		return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
	}

	// 6. Finalize
	logger.Info("Node scaled down successfully")
	scaler.Status.Phase = "Succeeded"
	scaler.Status.Message = "Node has been cordoned, drained, and terminated."
	if err := r.Status().Update(ctx, scaler); err != nil {
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *NodeScalerReconciler) SetupWithManager(mgr ctrl.Manager) error {
	// Get a clientset for the drain helper
	clientset, err := kubernetes.NewForConfig(mgr.GetConfig())
	if err != nil {
		return err
	}
	r.Clientset = clientset

	return ctrl.NewControllerManagedBy(mgr).
		For(&scalerv1.NodeScaler{}).
		Owns(&corev1.Node{}). // Watch Nodes owned by NodeScaler
		Complete(r)
}
'''

---

## 4. 工程卓越实践

### 4.1 单元与集成测试
`Kubebuilder` 集成了 `envtest`，它可以在本地启动一个临时的 `etcd` 和 `api-server` 用于测试。你应该为你的 `Reconcile` 逻辑编写详细的测试用例，覆盖所有成功和失败的路径。

'''bash
# 运行测试
make test
'''

### 4.2 日志与监控
- **结构化日志**：使用 `log.FromContext(ctx).WithValues("key", "value")` 提供结构化的日志，方便机器解析和查询。
- **Prometheus Metrics**：`controller-runtime` 默认暴露了 `/metrics` 端点。你可以定义自定义指标来监控控制器的行为，例如：
    '''go
    var nodesScaledDown = prometheus.NewCounter(
        prometheus.CounterOpts{
            Name: "nodes_scaled_down_total",
            Help: "Total number of nodes scaled down",
        },
    )
    // 在代码中调用
    nodesScaledDown.Inc()
    '''

### 4.3 RBAC 与安全
- **最小权限原则**：在 `+kubebuilder:rbac` 注释中，只授予控制器必需的最小权限。例如，如果只需要读取 Pods，就不要授予 `delete` 权限。
- **保护凭证**：访问云厂商 API 的凭证（如 AWS Key）绝不能硬编码在代码或镜像中。应使用环境变量、IAM 角色（推荐）或 Vault 等工具进行管理。

### 4.4 高可用性
为了避免单点故障，控制器应以多副本模式部署。`controller-runtime` 的 `Manager` 内置了**领导者选举 (Leader Election)** 机制。只需在 `main.go` 中启用它即可。

'''go
// main.go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    // ...
    LeaderElection:          true,
    LeaderElectionID:        "f8b9e5c5.example.com",
})
'''
启用后，即使部署多个控制器实例，也只有一个 Leader 会执行 `Reconcile` 逻辑，其他实例处于热备状态。

---

## 5. 部署与验证

`Kubebuilder` 已经通过 `make` 命令封装了所有部署步骤。

'''bash
# 1. 构建并推送 Docker 镜像
make docker-build docker-push IMG=your-repo/node-scaler-operator:v0.0.1

# 2. 部署到集群
make deploy IMG=your-repo/node-scaler-operator:v0.0.1
'''
此命令会自动应用 CRD、RBAC 规则和 Deployment。

**验证流程**：
1.  选择一个要缩容的节点，例如 `k8s-node-3`。
2.  创建一个 `NodeScaler` 实例来触发缩容：
    '''yaml
    # scale-down-node.yaml
    apiVersion: scaler.example.com/v1
    kind: NodeScaler
    metadata:
      name: scale-down-k8s-node-3
    spec:
      nodeName: "k8s-node-3"
    '''
    '''bash
    kubectl apply -f scale-down-node.yaml
    '''
3.  观察控制器日志和 `NodeScaler` 实例的状态变化：
    '''bash
    kubectl logs -f <controller-pod-name> -n node-scaler-operator-system
    kubectl get nodescaler scale-down-k8s-node-3 -o yaml
    '''
    你应该能看到 `status.phase` 依次变为 `Cordoning`, `Draining`, `Terminating`, `Succeeded`。

---

## 6. 结论

本文档从 Kubernetes 控制器的核心原理出发，深入探讨了其内部机制，并提供了一个遵循现代工程实践的、生产级的自定义控制器开发指南。通过使用 `controller-runtime` 和 `Kubebuilder` 等工具，开发者可以极大地提高开发效率，同时保证代码的健壮性和可维护性。真正的 Operator 开发是一项系统工程，需要综合考虑 API 设计、核心逻辑、测试、安全和可观测性等多个方面。

---

## 7. 参考资料

- [Kubernetes Controller Concepts](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Controller-Runtime Documentation](https://pkg.go.dev/sigs.k8s.io/controller-runtime)
- [Kubebuilder Book](https://book.kubebuilder.io/)
- [Client-Go Source Code](https://github.com/kubernetes/client-go)
- [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
