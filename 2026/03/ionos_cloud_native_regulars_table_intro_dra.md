---
theme: gopher.json
author: ""
paging: "@IONOS Cloud"
date: Mar 11, 2026
---

# An Introduction to Kubernetes Dynamic Resource Allocation

❯ $whoami

```yaml
Name: Gaurav Gahlot
Team: PaaS - Managed Kubernetes
```

---

## Device Plugin API

📌 The standard way to expose specialized hardware to Kubernetes workloads

```

```

```
~~~graph-easy
[GPU / FPGA / NIC] - device plugin -> [kubelet]
[kubelet] - allocate -> [Container]
~~~
```

---

## Device Plugin API — Limitations

```

```

| Limitation                 | Device Plugin API    | DRA                 |
| -------------------------- | -------------------- | ------------------- |
| Device sharing across Pods | ❌ Not supported     | ✅ Supported        |
| -------------------------- | -------------------- | ------------------- |
| Network-attached devices   | ❌ Not supported     | ✅ Supported        |
| -------------------------- | -------------------- | ------------------- |
| Fine-grained filtering     | ❌ Not possible      | ✅ CEL expressions  |
| -------------------------- | -------------------- | ------------------- |
| GPU partitioning (MIG)     | ❌ Coarse-grained    | ✅ Fine-grained     |
| -------------------------- | -------------------- | ------------------- |
| Allocation visibility      | ❌ Opaque            | ✅ Structured claim |
| -------------------------- | -------------------- | ------------------- |

---

## Dynamic Resource Allocation

📌 A flexible, vendor-neutral Kubernetes API for allocating specialized hardware

```yaml
stable: "since Kubernetes v1.34"
api: "resource.k8s.io/v1"
hardware:
  - GPUs
  - FPGAs
  - TPUs
  - High-performance NICs
  - ...
```

---

## Dynamic Resource Allocation — How It Works

```
~~~graph-easy
[GPU / Hardware] - ① detect -> [DRA Driver]
[DRA Driver] - ② advertise -> [API Server]
[User] - ③ request -> [API Server]
[API Server] - ④ trigger -> [Scheduler]
[Scheduler] - ⑤ allocate & run -> [Pod]
~~~
```

1. DRA driver on each node probes for devices and detects available hardware
2. Driver publishes discovered devices to the API server — visible cluster-wide
3. User submits a workload declaring a device requirement
4. API server triggers the scheduler, which finds a node with a matching free device
5. Scheduler binds the Pod to the node; Pod starts with access to the device

---

## The DRA Driver

📌 A vendor-written driver that may run as one or two parts

```
+------------------ Control Plane ------------------+
|                                                    |
|   kube-scheduler          kube-controller-manager  |
|   · reads ResourceSlices  · creates Claims from    |
|   · allocates Claims        Templates              |
|                                                    |
|   Driver Controller (Deployment) — if needed       |
|   · advanced cases: network-attached, cross-node   |
|                                                    |
+----------------------------------------------------+
                      ↑ API Server
+------------------ Node (×N) ----------------------+
|                                                    |
|   Driver Kubelet Plugin (DaemonSet) ←gRPC→ kubelet |
|   · publishes ResourceSlice                        |
|   · prepares device via CDI                        |
|   · cleans up on claim delete                      |
|                                                    |
+----------------------------------------------------+
```

📌 For node-local devices, only the kubelet plugin part is needed

---

## The Four Key Objects

```

```

| Object                    | Who        | Role                                        |
| ------------------------- | ---------- | ------------------------------------------- |
| **ResourceSlice**         | DRA driver | _"I have these devices on this node"_       |
| ------------------------- | ---------- | ------------------------------------------- |
| **DeviceClass**           | Admin      | _"This is a GPU category"_                  |
| ------------------------- | ---------- | ------------------------------------------- |
| **ResourceClaim**         | User       | _"I need one of those"_                     |
| ------------------------- | ---------- | ------------------------------------------- |
| **ResourceClaimTemplate** | User       | _"Give each Pod its own claim"_             |
| ------------------------- | ---------- | ------------------------------------------- |

---

## ResourceSlice

📌 The driver's way of saying _"here's what I have on this node"_

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
spec:
  driver: gpu.example.com # → device.driver in CEL
  nodeName: kind-worker
  pool:
    name: kind-worker
  devices:
    - name: gpu-0 # → device.name in CEL
      attributes:
        model: { string: LATEST-GPU-MODEL } # → device.attributes["model"]
        uuid: { string: gpu-4cbf87f3-... } # → device.attributes["uuid"]
        index: { int: 0 } # → device.attributes["index"]
      capacity:
        memory: { value: 80Gi } # → device.capacity["gpu.example.com"].memory
    # ... gpu-1 through gpu-8
```

📌 Device identity: `(driver name, pool name, device name)`

---

## DeviceClass

📌 The admin's way of _categorizing_ devices — using CEL

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu.example.com # ← referenced as deviceClassName in ResourceClaim
spec:
  selectors:
    - cel:
        # device.driver matches ResourceSlice.spec.driver
        expression: "device.driver == 'gpu.example.com'"
```

📌 Cluster-scoped — defined once, used by any `ResourceClaim`

---

## ResourceClaim

📌 The user's way of _requesting_ a device

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: example-resource-claim
spec:
  devices:
    requests:
      - name: example-gpu
        exactly:
          deviceClassName: gpu.example.com # ← DeviceClass.metadata.name
          allocationMode: ExactCount
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.capacity["gpu.example.com"].memory
                    == quantity("80Gi")
                                    # ↑ driver name (ResourceSlice.spec.driver)
                  # ↑ key from ResourceSlice.spec.devices[].capacity
```

---

## ResourceClaimTemplate

📌 Give _each Pod_ its own dedicated claim — note the double `spec:`

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
spec:
  spec: # outer: ResourceClaimTemplate
    devices: # inner: generated ResourceClaim
      requests:
        - name: req-0
          firstAvailable:
            - name: 80gi-gpu
              deviceClassName: gpu.example.com
              allocationMode: ExactCount
              count: 1
              selectors:
                - cel:
                    expression: >-
                      device.capacity["gpu.example.com"].memory
                        == quantity("80Gi")
```

---

## Using a ResourceClaim in a Pod

📌 Two connection points within the Pod spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ollama
spec:
  containers:
    - name: ollama
      image: alpine/ollama:0.11.10
      resources:
        claims:
          - name: gpu # ① must match spec.resourceClaims[].name
  resourceClaims:
    - name: gpu # ② internal alias — links container to claim
      resourceClaimName: example-resource-claim # ← ResourceClaim.metadata.name
```

📌 For per-Pod allocation, swap `resourceClaimName` for `resourceClaimTemplateName`

---

## Discovery — Devices Become Schedulable Resources

```
~~~graph-easy
[GPU on Node] -> [Kubelet Plugin]
[Kubelet Plugin] - creates -> [ResourceSlice]
[ResourceSlice] - stored in -> [API Server]
[API Server] - watched by -> [kube-scheduler]
~~~
```

📌 One `ResourceSlice` per node — visible to the scheduler cluster-wide

---

## Scheduling & Allocation

```
~~~graph-easy
[User] - creates -> [ResourceClaim]
[User] - creates -> [Pod refs claim]
[kube-scheduler] - reads -> [ResourceSlice]
[kube-scheduler] - selects node + device -> [ResourceClaim status]
[kube-scheduler] - binds -> [Pod refs claim]
~~~
```

📌 Scheduler writes the allocation into `ResourceClaim.status` — no vendor controller needed

---

## Device Preparation

📌 After scheduling, `kubelet` calls the plugin via gRPC

```sh
kubelet  →  NodePrepareResources()  →  Driver Kubelet Plugin
```

📌 The plugin writes a CDI spec — a file the container runtime reads at startup

```sh
GPU_DEVICE_7=...                           # defined inside the CDI spec
GPU_DEVICE_7_SHARING_STRATEGY=TimeSlicing  # defined inside the CDI spec
DRA_RESOURCE_DRIVER_NAME=gpu.example.com   # defined inside the CDI spec
```

📌 Plugin returns the CDI device ID → kubelet → container runtime → injects into container

---

## Device Sharing — One Claim, Two Pods

📌 Multiple Pods can reference the _same_ `ResourceClaim`

```
~~~graph-easy
[Pod: ollama] - references -> [example-resource-claim]
[Pod: ollama2] - references -> [example-resource-claim]
[example-resource-claim] - allocated to -> [gpu-7 (kind-worker)]
~~~
```

📌 The driver enforces the sharing strategy (e.g., TimeSlicing) at the CDI level

---

## Per-Pod Allocation — ResourceClaimTemplate + Job

📌 `kube-controller-manager` auto-creates one `ResourceClaim` per Pod

```
~~~graph-easy
[ResourceClaimTemplate] - auto-creates -> [Claim: Pod 1]
[ResourceClaimTemplate] - auto-creates -> [Claim: Pod 2]
[ResourceClaimTemplate] - auto-creates -> [Claim: Pod 3]
[ResourceClaimTemplate] - auto-creates -> [Claim: Pod 4]
[Claim: Pod 1] -> [gpu-1]
[Claim: Pod 2] -> [gpu-2]
[Claim: Pod 3] -> [gpu-3]
[Claim: Pod 4] -> [gpu-4]
~~~
```

📌 Each claim is deleted automatically when its Pod terminates

---

## DRA Drivers in the Wild

```

```

| Driver               | Vendor   | Devices                       |
| -------------------- | -------- | ----------------------------- |
| `gpu.nvidia.com`     | NVIDIA   | H100, A100, GB200             |
| -------------------- | -------- | ----------------------------- |
| `gpu.intel.com`      | Intel    | Arc, Data Center GPUs         |
| -------------------- | -------- | ----------------------------- |
| `k8s-gpu-dra-driver` | AMD      | Instinct, Radeon GPUs         |
| -------------------- | -------- | ----------------------------- |
| `dranet`             | Google   | High-perf NICs (RDMA, SR-IOV) |
| -------------------- | -------- | ----------------------------- |
| `dra-driver-cpu`     | k8s-sigs | NUMA-aware CPU allocation     |
| -------------------- | -------- | ----------------------------- |

---

## References

- [Kubernetes DRA Docs](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [Tutorial Repo](https://github.com/cloudnativeessentials/dra-tutorial)

---

## Thank you!

📌 Slides

- [Web](https://gauravgahlot.in/talks)
- [GitHub](https://github.com/gauravgahlot/talks)
