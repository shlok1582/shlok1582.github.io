---
layout: post
title: Demystifying Kubernetes Admission Controllers
subtitle: A Deep Dive tutorial
cover-img: /assets/img/custom-bg.png
tags: [kubernetes]
author: Shlok
---

I've been working with Kubernetes for a while now, and one of the areas that often seems like a black box is the admission controller system. If you've ever wondered how Kubernetes decides whether to accept or reject an API request, or how you can hook into that process to enforce custom policies, this post is for you.

In this deep dive, we'll explore:

- What admission controllers are and why they matter
- The difference between validating and mutating admission controllers
- How to write your own admission controller webhook
- Practical examples and gotchas I've encountered along the way

---

## What Are Admission Controllers?

Admission controllers are plugins that intercept API requests to the Kubernetes API server before they are persisted in etcd. They can mutate or validate the object in the request, enforcing custom policies beyond what the default Kubernetes roles and resources allow.

Think of them as gatekeepers that can:

- **Mutate** incoming requests, adding or modifying fields
- **Validate** requests, accepting or rejecting them based on custom logic

### Why Should You Care?

If you're running a production cluster, admission controllers can help you:

- Enforce security policies (e.g., prevent containers from running as root)
- Inject sidecar containers (e.g., for logging or monitoring)
- Ensure compliance with organizational policies

---

## Built-in Admission Controllers

Kubernetes comes with several built-in admission controllers that handle common tasks. Some of the most commonly used ones include:

- **NamespaceLifecycle**: Prevents modification of resources in terminating namespaces.
- **LimitRanger**: Enforces resource usage limits.
- **PodSecurityPolicy**: Controls security-sensitive aspects of the pod specification.

You can enable or disable these controllers via the `--enable-admission-plugins` flag on the API server.

---

## Validating vs. Mutating Admission Controllers

There are two types of admission controllers:

- **Mutating Admission Controllers**: These can modify the incoming object before it is stored.
- **Validating Admission Controllers**: These can accept or reject an object but cannot modify it.

Order matters: mutating controllers run before validating controllers.

---

## Writing a Custom Admission Controller Webhook

Sometimes, the built-in controllers aren't enough, and you need custom logic. This is where admission controller webhooks come into play.

### Overview

An admission webhook is an HTTPS endpoint that the API server calls with admission requests. Your webhook can then respond with instructions to allow, deny, or mutate the request.

### Setting Up the Webhook Server

Let's walk through setting up a simple validating webhook that prevents the creation of pods that don't have a specific label.

#### Prerequisites

- A Kubernetes cluster (I'm using minikube for this example)
- Go installed (version 1.16+)

#### Step 1: Writing the Webhook Server

Here's a basic Go program for the webhook:

```go
package main

import (
    "encoding/json"
    "fmt"
    "io/ioutil"
    "net/http"

    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
)

func serve(w http.ResponseWriter, r *http.Request) {
    body, err := ioutil.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "could not read request body", http.StatusBadRequest)
        return
    }

    var admissionReview admissionv1.AdmissionReview
    if err := json.Unmarshal(body, &admissionReview); err != nil {
        http.Error(w, "could not parse admission review", http.StatusBadRequest)
        return
    }

    podResource := admissionv1.GroupVersionResource{Group: "", Version: "v1", Resource: "pods"}
    if admissionReview.Request.Resource != podResource {
        http.Error(w, "expect resource to be pods", http.StatusBadRequest)
        return
    }

    var pod corev1.Pod
    if err := json.Unmarshal(admissionReview.Request.Object.Raw, &pod); err != nil {
        http.Error(w, "could not parse pod object", http.StatusBadRequest)
        return
    }

    allowed := true
    var result *admissionv1.AdmissionResponse
    if _, ok := pod.Labels["app"]; !ok {
        allowed = false
    }

    result = &admissionv1.AdmissionResponse{
        Allowed: allowed,
        UID:     admissionReview.Request.UID,
    }

    if !allowed {
        result.Result = &metav1.Status{
            Message: "pods must have an 'app' label",
        }
    }

    responseAdmissionReview := admissionv1.AdmissionReview{
        TypeMeta: admissionv1.TypeMeta{
            Kind:       "AdmissionReview",
            APIVersion: "admission.k8s.io/v1",
        },
        Response: result,
    }

    respBytes, err := json.Marshal(responseAdmissionReview)
    if err != nil {
        http.Error(w, "could not marshal response", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(respBytes)
}

func main() {
    http.HandleFunc("/validate", serve)
    fmt.Println("Starting webhook server...")
    err := http.ListenAndServeTLS(":8443", "/path/to/tls.crt", "/path/to/tls.key", nil)
    if err != nil {
        panic(err)
    }
}
```

#### Notes:

- The webhook server listens on `/validate` and handles admission review requests.
- It checks if the pod has an `app` label; if not, it rejects the request.

#### Step 2: Generating TLS Certificates

Kubernetes requires webhooks to use TLS. For development purposes, you can create a self-signed certificate.

```bash
openssl req -newkey rsa:2048 -nodes -keyout tls.key -x509 -days 365 -out tls.crt -subj "/CN=admission-webhook.default.svc"
```

#### Step 3: Deploying the Webhook Server

Create a Docker image of your webhook server and deploy it as a `Deployment` in your cluster.

```dockerfile
# Dockerfile
FROM golang:1.16-alpine
WORKDIR /app
COPY main.go .
COPY tls.crt /etc/webhook/tls.crt
COPY tls.key /etc/webhook/tls.key
RUN go build -o webhook .
CMD ["./webhook"]
```

Build and push the image to a registry accessible by your cluster.

#### Step 4: Configuring the Admission Webhook

Create a `ValidatingWebhookConfiguration`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-label-validator
webhooks:
  - name: pod-label-validator.example.com
    rules:
      - apiGroups: ["*"]
        apiVersions: ["*"]
        operations: ["CREATE"]
        resources: ["pods"]
    clientConfig:
      service:
        name: admission-webhook
        namespace: default
        path: "/validate"
      caBundle: <base64-encoded-tls.crt>
```

#### Step 5: Testing the Webhook

Try creating a pod without the `app` label:

```bash
kubectl run test-pod --image=nginx
```

You should see an error similar to:

```
Error from server: admission webhook "pod-label-validator.example.com" denied the request: pods must have an 'app' label
```

---

## Gotchas and Practical Tips

### Certificate Management

One of the tricky parts is managing certificates, especially in production. You'll need to ensure that the API server trusts your webhook's certificate.

- Use Kubernetes' built-in certificate signing requests (CSRs) to automate certificate management.
- Consider using cert-manager to handle certificate issuance and renewal.

### Failure Policy

In your webhook configuration, you can set `failurePolicy` to `Ignore` or `Fail`.

- **Fail**: The API server will reject requests if the webhook fails.
- **Ignore**: The API server will allow requests even if the webhook fails.

Be cautious with `Fail`; if your webhook is down, it could block deployments.

### Timeout Configuration

Set an appropriate timeout for your webhook to prevent hanging requests. The default is 30 seconds, but shorter timeouts (e.g., 5 seconds) are often sufficient.

---

## Real-World Use Cases

In my experience, admission controllers are invaluable for:

- **Security Enforcement**: Preventing deployment of images from untrusted registries.
- **Resource Quotas**: Enforcing limits on CPU and memory usage beyond what's possible with LimitRanges.
- **Policy Enforcement**: Ensuring all resources have necessary annotations or labels for compliance.
