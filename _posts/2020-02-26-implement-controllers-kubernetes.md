---
layout: post
title: Implementing Custom Controllers and Operators in Kubernetes
subtitle: A Deep Dive tutorial
cover-img: /assets/img/custom-bg.png
tags: [kubernetes]
author: Shlok
---

## Introduction

Kubernetes has become the de facto standard for container orchestration, providing a powerful platform for deploying, scaling, and managing containerized applications. While Kubernetes offers a rich set of built-in resources and controllers, there are scenarios where you need to extend its functionality to meet custom requirements. This is where **Custom Controllers** and **Operators** come into play.

In this post, we'll take an in-depth look at how to implement custom controllers and operators in Kubernetes. We'll explore the underlying architecture, delve into the reconciliation loop, and provide code examples to illustrate key concepts.

---

## Table of Contents

1. [Understanding Kubernetes Controllers](#understanding-kubernetes-controllers)
2. [Custom Resource Definitions (CRDs)](#custom-resource-definitions-crds)
3. [Building a Custom Controller](#building-a-custom-controller)
   - [Prerequisites](#prerequisites)
   - [Project Setup](#project-setup)
   - [Controller Logic](#controller-logic)
4. [Deep Dive into the Reconciliation Loop](#deep-dive-into-the-reconciliation-loop)
5. [Implementing an Operator](#implementing-an-operator)
6. [Advanced Topics](#advanced-topics)
   - [Concurrency and Thread Safety](#concurrency-and-thread-safety)
   - [Error Handling and Retries](#error-handling-and-retries)
   - [Performance Considerations](#performance-considerations)
7. [Conclusion](#conclusion)

---

## Understanding Kubernetes Controllers

At the core of Kubernetes' self-healing capabilities are **controllers**. A controller is a control loop that watches the state of your cluster through the Kubernetes API and makes or requests changes where needed. Controllers ensure that the current state of the cluster matches the desired state.

### Built-in Controllers

Kubernetes comes with several built-in controllers, such as:

- **Deployment Controller**: Manages ReplicaSets to ensure the desired number of pods.
- **Job Controller**: Manages batch jobs that run to completion.
- **Node Controller**: Manages node status and responds to node failures.

### Custom Controllers

Custom controllers allow you to define custom behavior for your applications or infrastructure. They can manage both built-in Kubernetes resources and **Custom Resources** that you define.

---

## Custom Resource Definitions (CRDs)

**Custom Resource Definitions (CRDs)** enable you to extend the Kubernetes API with your own resource types. When you create a CRD, Kubernetes API Server starts serving the new custom resource's endpoints, allowing you to perform CRUD operations on instances of your custom resource.

### Defining a CRD

Here's an example of a CRD YAML definition:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: widgets
    singular: widget
    kind: Widget
    shortNames:
      - wd
```

After applying this CRD, you can create custom resources of type `Widget`.

---

## Building a Custom Controller

To build a custom controller, we'll use the [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) library, which simplifies the process of writing controllers by providing high-level abstractions.

### Prerequisites

- Go programming language installed (version 1.16 or higher)
- Basic understanding of Kubernetes APIs and Go modules

### Project Setup

Initialize a new Go module:

```bash
mkdir widget-controller
cd widget-controller
go mod init example.com/widget-controller
```

Import the necessary packages in your `main.go`:

```go
import (
    "context"
    "flag"
    "os"

    "example.com/widget-controller/controllers"
    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
)

var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    // Add your custom resource scheme here
    // utilruntime.Must(widgetv1.AddToScheme(scheme))
}
```

### Controller Logic

Create a new controller under `controllers/widget_controller.go`:

```go
package controllers

import (
    "context"

    "example.com/widget-controller/api/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"
)

type WidgetReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// Reconcile is the main logic of the controller
func (r *WidgetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // Fetch the Widget instance
    var widget v1.Widget
    if err := r.Get(ctx, req.NamespacedName, &widget); err != nil {
        logger.Error(err, "unable to fetch Widget")
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Implement reconciliation logic here

    return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *WidgetReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&v1.Widget{}).
        Complete(r)
}
```

In your `main.go`, set up the manager and start the controller:

```go
func main() {
    // ... [Initialization code] ...

    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }

    if err = (&controllers.WidgetReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "Widget")
        os.Exit(1)
    }

    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

---

## Deep Dive into the Reconciliation Loop

The **Reconciliation Loop** is the heart of a Kubernetes controller. It's responsible for moving the current state of the cluster towards the desired state.

1. **Watch**: The controller watches for events related to the resources it's interested in (e.g., create, update, delete events for `Widget` resources).

2. **Queue**: Events are added to a work queue to ensure they're processed in order and to handle retries.

3. **Reconcile Function**: The controller processes each item in the queue by executing the reconcile function.

### Implementing Reconciliation Logic

Within the `Reconcile` method, you perform the following steps:

- **Fetch the Custom Resource**: Retrieve the latest state of the resource from the API server.

- **Compare Desired and Current State**: Determine what changes are necessary to align the current state with the desired state specified in the custom resource.

- **Perform Actions**: Create, update, or delete Kubernetes resources as needed.

### Example Reconciliation Logic

```go
func (r *WidgetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // Fetch the Widget instance
    var widget v1.Widget
    if err := r.Get(ctx, req.NamespacedName, &widget); err != nil {
        logger.Error(err, "unable to fetch Widget")
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Define the desired Deployment based on the Widget spec
    desiredDeployment := r.constructDeployment(&widget)

    // Fetch the current Deployment
    var currentDeployment appsv1.Deployment
    err := r.Get(ctx, types.NamespacedName{
        Name:      desiredDeployment.Name,
        Namespace: desiredDeployment.Namespace,
    }, &currentDeployment)

    if err != nil && apierrors.IsNotFound(err) {
        // Deployment doesn't exist, create it
        if err := r.Create(ctx, desiredDeployment); err != nil {
            logger.Error(err, "failed to create Deployment")
            return ctrl.Result{}, err
        }
        logger.Info("created Deployment", "name", desiredDeployment.Name)
    } else if err != nil {
        // Error fetching Deployment
        return ctrl.Result{}, err
    } else {
        // Deployment exists, update if necessary
        if !reflect.DeepEqual(currentDeployment.Spec, desiredDeployment.Spec) {
            currentDeployment.Spec = desiredDeployment.Spec
            if err := r.Update(ctx, &currentDeployment); err != nil {
                logger.Error(err, "failed to update Deployment")
                return ctrl.Result{}, err
            }
            logger.Info("updated Deployment", "name", currentDeployment.Name)
        }
    }

    return ctrl.Result{}, nil
}
```

---

## Implementing an Operator

An **Operator** extends a controller by encoding domain-specific knowledge to automate the management of complex applications. Operators use custom controllers to manage custom resources, encapsulating operational logic.

### Operator Frameworks

- **Operator SDK**: Simplifies the process of building operators by providing scaffolding and tools.
- **Kubebuilder**: A framework for building Kubernetes APIs using custom resource definitions.

### Example: Creating an Operator with Operator SDK

1. **Initialize the Project**:

   ```bash
   operator-sdk init --domain=example.com --repo=example.com/widget-operator
   ```

2. **Create a New API**:

   ```bash
   operator-sdk create api --group=example --version=v1 --kind=Widget --resource --controller
   ```

3. **Implement the Controller Logic**: Edit the generated controller file under `controllers/`.

4. **Build and Deploy the Operator**:

   ```bash
   make docker-build docker-push IMG=your-repo/widget-operator:tag
   make deploy IMG=your-repo/widget-operator:tag
   ```

---

## Advanced Topics

### Concurrency and Thread Safety

Controllers can process multiple events concurrently. Use work queues and synchronization mechanisms to handle concurrency.

- **Worker Pools**: Limit the number of concurrent reconciliations.
- **Mutexes**: Protect shared resources within the controller.

### Error Handling and Retries

- **Requeueing**: Return an error or a `RequeueAfter` result to retry reconciliation.
- **Idempotency**: Ensure that your reconcile logic can handle retries without side effects.

### Performance Considerations

- **Caching**: Use informers to cache resource states and reduce API calls.
- **Rate Limiting**: Control the rate of event processing to prevent overloading the API server.

---

## References

- [Kubernetes Documentation: Extending Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/)
- [Controller Runtime Library](https://github.com/kubernetes-sigs/controller-runtime)
- [Operator SDK](https://sdk.operatorframework.io/)
- [Kubebuilder](https://book.kubebuilder.io/)

---
