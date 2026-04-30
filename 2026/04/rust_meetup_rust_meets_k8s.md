---
author: ""
paging: "@Rust Berlin"
date: Apr 30, 2026
---

# Rust meets Kubernetes

---

## $whoami

```yaml
Name: Gaurav Gahlot
Role: Staff Software Engineer, IONOS Cloud
X: _gauravgahlot
Web: https://gauravgahlot.in/
GitHub: gauravgahlot
OSS: Akri (maintainer, CNCF Sandbox), kube-dra
Community: Cloud Native Berlin (organizer)
```

---

## The Kubernetes Ecosystem

The entire operator ecosystem assumes Go.

client-go, controller-runtime, KubeBuilder — all Go.

```

```

But Rust has a real and growing foothold.

---

## Memory-Safe ≠ CVE-Free

The difference isn't whether you patch — it's what you patch.

```
+---------- Go: stdlib monolith -----------+
|   net/http   crypto/tls   encoding/json  |
|   x/net   x/crypto   ...                 |
+------------------------------------------+
   CVE in net/http → Go 1.21.3, 1.20.10
   rebuild EVERY Go binary in the cluster

```

```

+--------- Rust: ecosystem crates ---------+
|   hyper   h2   rustls   tokio   serde    |
|   ...                                    |
+------------------------------------------+
   CVE in h2 → bump the h2 crate
   rebuild only binaries that link it
```

---

## What's a Controller?

A program that watches a resource and drives the cluster toward what you declared.

```
~~~graph-easy
[API Server] - watch -> [Controller]
[Controller] - reconcile / create / update -> [API Server]
~~~
```

📌 Pattern: observe → compare → act → repeat.

📌 Custom controllers extend Kubernetes with your domain — your CRDs, your reconcile logic.

---

## The Framework Caches the Cluster

Controllers are stateless. The framework is not.

- kube-runtime keeps an in-memory cache of every resource the controller watches — the informer store.
- Reconcile reads from it on every event.

```

~~~graph-easy
[API Server] - watch -> [Informer Cache]
[Informer Cache] - read -> [reconcile()]
[Informer Cache] - Go GC scans every cycle -> [p99 spike]
~~~
```

In Rust, memory is freed when the owner drops. No scanning. No pauses.

---

## kube-rs — Rust's Answer to client-go

```
kube
├── kube-client    HTTP client, Api<T>, Config
├── kube-core      Types, traits, parameters
├── kube-runtime   Controller, watcher, reflector, finalizer
└── kube-derive    #[derive(CustomResource)]
```

One dependency, batteries included.

---

## Three Lines to Get Started

```go
[dependencies]
kube = { version = "3", features = ["runtime", "derive"] }
k8s-openapi = { version = "0.27", features = ["latest"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
```

📌 k8s-openapi gives you typed Pod, Node, ConfigMap, Deployment, etc.

📌 kube gives you Client, Api, Controller, watcher, CustomResource.

---

## Example: Connect and List Pods

```go
use k8s_openapi::api::core::v1::Pod;
use kube::{Api, api::ListParams};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // reads $KUBECONFIG, ~/.kube/config, or in-cluster service account
    let client = kube::Client::try_default().await?;

    // typed handle — this Api will only speak to Pods
    let pods: Api<Pod> = Api::namespaced(client, "kube-system");

    for pod in pods.list(&ListParams::default()).await? {
        let name = pod.metadata.name.unwrap();
        let phase = pod.status.unwrap().phase.unwrap();
        println!("{name} — {phase}");
    }

    Ok(())
}
```

---

## Example: Watch Pod Events

```go
use kube::{ResourceExt, runtime::{watcher, WatchStreamExt}};
use futures::TryStreamExt;

let pods: Api<Pod> = Api::namespaced(client, "default");

// watcher() returns a real Stream<Item = Result<Event<Pod>>>
watcher(pods, watcher::Config::default())
    .default_backoff()    // auto-retry with exponential backoff on disconnect
    .touched_objects()    // flatten Event<Pod> → just the changed Pods
    .try_for_each(|pod| async move {
        println!("changed: {}", pod.name_any());
        Ok(())
    })
    .await?;
```

📌 Kubernetes pushes changes to you. You don't poll.

📌 Async streams — composable, back-pressure aware.

---

## Example: A Minimal Controller

```go
// imports omitted: Arc, Duration, ConfigMap, Action, ResourceExt
struct Context { client: kube::Client }

// called on every resource change — must be idempotent
async fn reconcile(
    obj: Arc<ConfigMap>, ctx: Arc<Context>     // Arc → shared across async tasks
) -> Result<Action, kube::Error> {
    println!("reconciling: {}", obj.name_any());
    Ok(Action::requeue(Duration::from_secs(300)))   // re-run in 5 min
}

// runs when reconcile() returns Err — decides retry strategy
fn error_policy(
    _obj: Arc<ConfigMap>, err: &kube::Error, _ctx: Arc<Context>
) -> Action {
    eprintln!("error: {err}");
    Action::requeue(Duration::from_secs(5))         // faster retry on error
}
```

---

## Running the Controller

```go
// imports omitted: Arc, Client, Api, ConfigMap, Controller, watcher, futures::StreamExt
let client = Client::try_default().await?;
let cms: Api<ConfigMap> = Api::namespaced(client.clone(), "default");

Controller::new(cms, watcher::Config::default())
    // framework: watch + work queue + dedup + concurrent reconciles
    .run(reconcile, error_policy, Arc::new(Context { client }))
    .for_each(|res| async move {
        // stream yields one item per completed reconcile
        match res {
            Ok((obj, _)) => println!("reconciled: {obj:?}"),
            Err(e) => eprintln!("reconcile failed: {e}"),
        }
    })
    .await;
```

📌 The framework handles watch, queue, dedup, backoff.

📌 You write the reconcile function. The framework does the rest.

---

## Honest Trade-offs

| Aspect              | Go (client-go)       | Rust (kube-rs)         |
| ------------------- | -------------------- | ---------------------- |
| Feature parity      | Complete reference   | Some gaps              |
| ------------------- | -------------------- | ---------------------- |
| Learning curve      | Goroutines, simple   | Async + ownership      |
| ------------------- | -------------------- | ---------------------- |
| Memory              | GC scans the heap    | Drops free memory      |
| ------------------- | -------------------- | ---------------------- |
| Data races          | Possible at runtime  | Caught at compile      |
| ------------------- | -------------------- | ---------------------- |

```

```

📌 Mature enough for production today — but expect to invest in async Rust upfront.

---

## Rust in the Cloud-Native Stack

```

```

**TiKV** — CNCF Graduated, distributed transactional KV store (TiDB's storage layer)

**Linkerd** — data-plane proxy (linkerd2-proxy) in Rust, runs in every pod

**Podman** — Rust networking stack (netavark, aardvark-dns)

**Vector** — observability data pipeline (logs, metrics, traces)

**Spin / SpinKube** — Wasm app framework + CNCF Sandbox K8s integration

**youki** — CNCF Sandbox, container runtime (runc replacement)

**Akri** — CNCF Sandbox, IoT device management framework

---

# Thank you!

```yaml
Slides: https://gauravgahlot.in/talks
```
