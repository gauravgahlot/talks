# Try It Tonight: What `resourceVersion` Really Is

Every object you get back from Kubernetes carries a `resourceVersion`. This guide shows you where that
number comes from, on your own laptop. It goes with the DevFest talk _What resourceVersion Really Is:
Kubernetes, etcd and MVCC_, and the [slides](./devfest_what_resourceversion_really_is.md) are in this
folder.

You need two terms first.

- **kind** (Kubernetes in Docker) runs a whole Kubernetes cluster inside one Docker container.
- **etcd** is the database where Kubernetes stores every object. Normally only the API server talks to
  it. Here, `etcdctl` talks to it directly, so you can see what Kubernetes sees.

The outputs below are real. They come from one run on 12 November 2026, on macOS with an Apple M5 Pro,
kind v0.32.0, Kubernetes v1.36.1, etcd 3.6.8 and etcdctl 3.7.1. Revision and `resourceVersion`
numbers are left out, because yours will never match. The text says what to look for instead.

---

## What you need

- **Docker**, running.
- **[kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)**
- **[kubectl](https://kubernetes.io/docs/tasks/tools/)**, the command-line client for Kubernetes.
- **`etcdctl`**, which ships with [etcd](https://github.com/etcd-io/etcd/releases). On macOS,
  `brew install etcd` installs it.
- **[jq](https://jqlang.org/download/)**, for picking fields out of JSON.

Two things trip people up.

- **Port 2379 must be free.** It is etcd's port, and the cluster publishes it on your laptop.
  `lsof -nP -iTCP:2379 -sTCP:LISTEN` should print nothing.
- **Quote every `jq` filter.** zsh, the default shell on macOS, reads an unquoted `.kvs[0]` as a file
  pattern and fails with `no matches found`.

---

## Set up the cluster

Create a working directory and the cluster config, then the cluster.

```bash
mkdir -p ~/rv-demo && cd ~/rv-demo
cat > kind.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 2379          # etcd's client port, so etcdctl on the laptop can reach it
        hostPort: 2379
        listenAddress: "127.0.0.1"
kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        - name: etcd-compaction-interval
          value: "0"                 # no automatic compaction: step 7 compacts by hand
EOF
kind create cluster --name rv-demo --config kind.yaml --kubeconfig ./rv-demo.kubeconfig
```

- `extraPortMappings` publishes etcd's port 2379 on your laptop's `127.0.0.1`. That is what lets
  `etcdctl` talk to etcd directly.
- `etcd-compaction-interval: "0"` switches off a cleanup that Kubernetes normally runs every 5
  minutes. Without it, the history you replay in step 6 could already be gone. Step 7 runs the
  cleanup by hand.
- `--kubeconfig ./rv-demo.kubeconfig` writes the cluster's credentials to a file of its own, so your
  usual kubeconfig is not touched.

Copy etcd's certificates out of the cluster, then point both tools at it.

```bash
mkdir -p pki
docker cp rv-demo-control-plane:/etc/kubernetes/pki/etcd/ca.crt pki/ca.crt
docker cp rv-demo-control-plane:/etc/kubernetes/pki/apiserver-etcd-client.crt pki/client.crt
docker cp rv-demo-control-plane:/etc/kubernetes/pki/apiserver-etcd-client.key pki/client.key

export KUBECONFIG=$PWD/rv-demo.kubeconfig
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=$PWD/pki/ca.crt
export ETCDCTL_CERT=$PWD/pki/client.crt
export ETCDCTL_KEY=$PWD/pki/client.key
```

- `ca.crt` is the authority that signed etcd's certificates. `etcdctl` uses it to trust etcd.
- `apiserver-etcd-client` is the identity the API server itself uses to talk to etcd. We borrow it.
- `KUBECONFIG` makes `kubectl` in this terminal see the kind cluster and nothing else.
- The four `ETCDCTL_*` variables mean no `etcdctl` command below needs a flag to connect.

Check both paths.

```bash
kubectl config get-contexts -o name
etcdctl endpoint status -w table
```

```text
kind-rv-demo
┌────────────────────────┬─────────────────┬─────────┬─────────────────┬─────────┬────────┬───────────────────────┬────────┬───────────┬
│        ENDPOINT        │       ID        │ VERSION │ STORAGE VERSION │ DB SIZE │ IN USE │ PERCENTAGE NOT IN USE │ QUOTA  │ IS LEADER │
├────────────────────────┼─────────────────┼─────────┼─────────────────┼─────────┼────────┼───────────────────────┼────────┼───────────┼
│ https://127.0.0.1:2379 │ e58c878e0e01014 │   3.6.8 │           3.6.0 │  918 kB │ 913 kB │                    1% │ 2.1 GB │      true │
└────────────────────────┴─────────────────┴─────────┴─────────────────┴─────────┴────────┴───────────────────────┴────────┴───────────┴
```

The table is cut on the right. If you see your own etcd's version and size, `etcdctl` is connected.

---

## 1. Your objects are keys

A ConfigMap is a small Kubernetes object that holds settings for an app. Create one, and list the
ConfigMaps in the `default` namespace.

```bash
kubectl create configmap app-config --from-literal=color=blue
kubectl get configmaps -n default
```

```text
configmap/app-config created
NAME               DATA   AGE
app-config         1      0s
kube-root-ca.crt   1      40s
```

Now ask etcd for the same thing.

```bash
etcdctl get /registry/configmaps/default/ --prefix --keys-only
etcdctl get /registry --prefix --count-only -w fields | grep Count
```

```text
/registry/configmaps/default/app-config

/registry/configmaps/default/kube-root-ca.crt

"Count" : 345
```

- Every Kubernetes object is one key in etcd, shaped `/registry/<resource>/<namespace>/<name>`.
- `--prefix --keys-only` returns every key that starts with `/registry/configmaps/default/`. It finds
  the same two objects. Listing objects in Kubernetes is a scan over keys that share a prefix.
- `--count-only -w fields` counts keys without fetching them. A fresh cluster holds 345.

---

## 2. One counter for everything

etcd keeps one counter for the whole database, called the **revision**. Watch it on a cluster that
nobody is touching.

```bash
etcdctl endpoint status -w json | jq '.[0].Status.header.revision'
sleep 10
etcdctl endpoint status -w json | jq '.[0].Status.header.revision'
```

It moved a few times in 10 seconds, and you did nothing. Now write a key of your own, outside the part
of etcd that Kubernetes uses.

```bash
etcdctl put /not-kubernetes/hello world -w json | jq '.header.revision'
etcdctl get /not-kubernetes/hello -w json | jq '.kvs[0].mod_revision'
```

Kubernetes will never read that key. It still took the next number from the same counter. So who
made the other writes?

```bash
etcdctl watch --prefix /registry | grep -a '^/registry'
```

Press Ctrl-C after about 10 seconds.

```text
/registry/leases/kube-system/kube-controller-manager
/registry/leases/kube-system/kube-scheduler
/registry/masterleases/172.18.0.2
/registry/peerserverleases/apiserver-4n66ybokvureugorpvxi27ujj4
/registry/leases/kube-system/kube-controller-manager
/registry/leases/kube-system/apiserver-4n66ybokvureugorpvxi27ujj4
/registry/leases/kube-system/kube-scheduler
/registry/leases/kube-system/kube-controller-manager
/registry/leases/kube-node-lease/rv-demo-control-plane
/registry/leases/kube-system/kube-scheduler
/registry/leases/kube-system/kube-controller-manager
/registry/leases/kube-system/kube-scheduler
/registry/leases/kube-system/kube-controller-manager
/registry/leases/kube-system/kube-scheduler
```

- A **watch** is a live stream of changes. `etcdctl watch --prefix /registry` streams every change to
  a Kubernetes object. Each change prints a key and a binary value, so `grep -a '^/registry'` keeps
  only the keys.
- Every write here is a **lease** renewal. A lease is a small object that a component rewrites every
  few seconds, to show it is still alive. The scheduler and the controller manager renew theirs every
  2 seconds, and the node renews its heartbeat every 10.

---

## 3. Four numbers on every key

```bash
kubectl get configmap app-config -o jsonpath='{.metadata.resourceVersion}{"\n"}'
etcdctl get /registry/configmaps/default/app-config -w json \
  | jq -c '{revision: .header.revision, create_revision: .kvs[0].create_revision, mod_revision: .kvs[0].mod_revision, version: .kvs[0].version}'
CREATED=$(etcdctl get /registry/configmaps/default/app-config -w json | jq '.kvs[0].create_revision')
```

`CREATED` keeps the creation revision for steps 6 and 7. Now update the ConfigMap twice.

```bash
kubectl patch configmap app-config --type merge -p '{"data":{"color":"green"}}'
kubectl patch configmap app-config --type merge -p '{"data":{"color":"red"}}'
kubectl get configmap app-config -o jsonpath='{.metadata.resourceVersion}{"\n"}'
etcdctl get /registry/configmaps/default/app-config -w json \
  | jq -c '{revision: .header.revision, create_revision: .kvs[0].create_revision, mod_revision: .kvs[0].mod_revision, version: .kvs[0].version}'
```

- `revision` is the counter at the moment of the read. It is already ahead of the object, because
  other writes happened in between.
- `create_revision` is when this key was created. It stays the same through both updates.
- `mod_revision` is the last write to this key. **It is the `resourceVersion`**: it matches what
  `kubectl` printed, before and after the updates.
- `version` counts this key's own writes, so 3. It is the number most people think `resourceVersion`
  is, and Kubernetes never shows it.

### Bonus: `generation`

Some objects carry a third counter. The **spec** is the part of an object that says what you want,
and `generation` counts changes to it.

```bash
kubectl create deployment web --image=registry.k8s.io/pause:3.10
sleep 6
kubectl get deployment web -o jsonpath='generation={.metadata.generation} resourceVersion={.metadata.resourceVersion}{"\n"}'
kubectl scale deployment web --replicas=3
sleep 6
kubectl get deployment web -o jsonpath='generation={.metadata.generation} resourceVersion={.metadata.resourceVersion}{"\n"}'
kubectl label deployment web team=demo
sleep 3
kubectl get deployment web -o jsonpath='generation={.metadata.generation} resourceVersion={.metadata.resourceVersion}{"\n"}'
kubectl get configmap app-config -o jsonpath='generation=[{.metadata.generation}]{"\n"}'
```

- Scaling changed the spec, so `generation` went to 2. A label is not part of the spec, so it stayed.
- `resourceVersion` moved on every write, including the status updates that Kubernetes made itself.
- A ConfigMap has no spec, and no `generation`.
- The image is the pause image that kind already has, so nothing is downloaded.

---

## 4. A noisy neighbour

Write down the counter before the noise starts. Step 7 needs it.

```bash
BEFORE=$(etcdctl endpoint status -w json | jq '.[0].Status.header.revision'); echo $BEFORE
```

Now start a neighbour that writes a lot. It is a Job, a Kubernetes object that runs a program until
it finishes, and this program never finishes on its own.

```bash
cat > noisy-producer.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: noisy
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: noisy
  namespace: noisy
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-writer
  namespace: noisy
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-writer
  namespace: noisy
subjects:
  - kind: ServiceAccount
    name: noisy
    namespace: noisy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: configmap-writer
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: noise
  namespace: noisy
---
apiVersion: batch/v1
kind: Job
metadata:
  name: noisy-producer
  namespace: noisy
spec:
  activeDeadlineSeconds: 1800  # stops itself after 30 minutes if you forget it
  backoffLimit: 0
  template:
    spec:
      serviceAccountName: noisy
      restartPolicy: Never
      containers:
        - name: writer
          image: curlimages/curl:8.11.1
          command: ["/bin/sh", "-c"]
          args:
            - |
              SA=/var/run/secrets/kubernetes.io/serviceaccount
              TOKEN=$(cat $SA/token)
              URL=https://kubernetes.default.svc/api/v1/namespaces/noisy/configmaps/noise
              i=0
              while true; do
                i=$((i+1))
                curl -s -o /dev/null --cacert $SA/ca.crt \
                  -H "Authorization: Bearer $TOKEN" \
                  -H "Content-Type: application/merge-patch+json" \
                  -X PATCH "$URL" -d "{\"data\":{\"n\":\"$i\"}}"
              done
EOF
kubectl apply -f noisy-producer.yaml
kubectl -n noisy wait --for=condition=Ready pod -l job-name=noisy-producer --timeout=120s
```

```text
namespace/noisy created
serviceaccount/noisy created
role.rbac.authorization.k8s.io/configmap-writer created
rolebinding.rbac.authorization.k8s.io/configmap-writer created
configmap/noise created
job.batch/noisy-producer created
pod/noisy-producer-9tlv4 condition met
```

- The Job runs one container that patches the ConfigMap `noise` in a loop, using `curl` against the
  API server. Every patch is a real Kubernetes write.
- The `ServiceAccount`, `Role` and `RoleBinding` give it permission to patch ConfigMaps in its own
  namespace, and nothing else.
- The first run downloads the `curlimages/curl` image, so give it a minute.

How noisy is it?

```bash
etcdctl endpoint status -w json | jq '.[0].Status.header.revision'
sleep 10
etcdctl endpoint status -w json | jq '.[0].Status.header.revision'
```

That is ~1,400 writes in 10 seconds, about 146 a second. Now write `app-config` once more.

```bash
kubectl get configmap app-config -o jsonpath='{.metadata.resourceVersion}{"\n"}'
kubectl patch configmap app-config --type merge -p '{"data":{"color":"purple"}}'
kubectl get configmap app-config -o jsonpath='{.metadata.resourceVersion}{"\n"}'
etcdctl get /registry/configmaps/default/app-config -w json | jq '.kvs[0].version'
```

- The two `resourceVersion` values are thousands apart, from a single write.
- `version` prints 4: blue, green, red, and now purple. It is the only number here that counts
  `app-config`'s own writes.
- The gap is the neighbour's writes, which moved the shared counter in between. Picture an etcd
  shared by many clusters, running for a long time, and you get numbers in the tens of billions.

Leave the noisy producer running. Steps 7 and 8 need it.

---

## 5. Two writers, one stale

Take a copy of the ConfigMap, let someone else write first, then send your copy back.

```bash
kubectl get configmap app-config -o yaml > app-config.yaml
grep resourceVersion app-config.yaml
kubectl patch configmap app-config --type merge -p '{"data":{"color":"yellow"}}'
kubectl replace -f app-config.yaml
```

```text
configmap/app-config patched
Error from server (Conflict): error when replacing "app-config.yaml": Operation cannot be fulfilled on configmaps "app-config": the object has been modified; please apply your changes to the latest version and try again
```

- `app-config.yaml` is your copy, holding the `resourceVersion` that `grep` printed first.
- The patch plays someone else writing first. It moves the object to a newer revision.
- `kubectl replace` sends your copy back with its `resourceVersion`. The API server asks etcd to write
  only if `mod_revision` still equals it. It doesn't, so nothing is written, and you get
  `409 Conflict`.
- It is the same idea as a rejected `git push`. The content of your copy never mattered, only how old
  it was.

---

## 6. Reading the past

etcd never overwrites a value. Each write adds a new version and keeps the old ones. That is called
**MVCC**, multi-version concurrency control. Delete the ConfigMap, then read it anyway.

```bash
kubectl delete configmap app-config
etcdctl get /registry/configmaps/default/app-config
etcdctl get /registry/configmaps/default/app-config --rev=$CREATED --print-value-only | strings
```

```text
configmap "app-config" deleted from default namespace
ConfigMap
app-config
default"
*$1a7289d6-9f33-4b23-a7a8-c3ab0146c7eb2
kubectl-create
Update
FieldsV1:"
 {"f:data":{".":{},"f:color":{}}}B
color
blue
```

- After the delete, a plain `get` prints nothing.
- `--rev=$CREATED` reads the key as it was at the revision you created it. The stored object is
  binary, so `strings` pulls out the readable parts. The original `color: blue` is still there.

Now replay the object's whole life.

```bash
etcdctl watch /registry/configmaps/default/app-config --rev=$CREATED -w json \
  | jq -c '.Events[] | {event: (if .type == 1 then "DELETE" else "PUT" end), version: (.kv.version // 0)}'
```

Press Ctrl-C once the output stops.

```text
{"event":"PUT","version":1}
{"event":"PUT","version":2}
{"event":"PUT","version":3}
{"event":"PUT","version":4}
{"event":"PUT","version":5}
{"event":"DELETE","version":0}
```

- A watch that starts from an old revision first replays every change since then, in order: created
  blue, patched green, red, purple and yellow, then deleted.
- The `DELETE` is a **tombstone**, a marker that says the key was deleted at that revision. It is a
  write of its own, and it took a number from the counter like any other.
- Add `mod_revision: .kv.mod_revision,` to the filter to see each event's revision. From the purple
  write on, they jump by thousands, because the noisy producer was writing in between.
- etcd's JSON leaves the event type out for a PUT, so the filter fills it in.

---

## 7. 410 Gone

Keeping every version forever would fill the disk. So old versions get thrown away, which is called
**compaction**. Kubernetes normally does it every 5 minutes, and you switched that off at setup. Do it
by hand now.

```bash
REV=$(etcdctl endpoint status -w json | jq '.[0].Status.header.revision'); echo $REV
etcdctl compact $REV
etcdctl get /registry/configmaps/default/app-config --rev=$CREATED
etcdctl watch /registry/configmaps/default/app-config --rev=$CREATED
```

```text
Error: etcdserver: mvcc: required revision has been compacted
watch was canceled (etcdserver: mvcc: required revision has been compacted)
Error: watch is canceled by the server
```

- `etcdctl compact` discards every version older than the current revision. Before the errors, you
  see that revision twice: once from `echo`, once in `compacted revision`.
- The history you replayed in step 6 is gone. etcd refuses to read or watch from before that point.
- etcdctl also prints a JSON retry warning above the first `Error:` line. It is left out here.

Now act like a program that watched ConfigMaps, lost its connection during the noise, and comes back
asking to continue from `$BEFORE`.

```bash
kubectl get --raw "/api/v1/namespaces/default/configmaps?watch=1&resourceVersion=$BEFORE&timeoutSeconds=5"
```

```text
{"type":"ERROR","object":{"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"too old resource version: <BEFORE> (<N>)","reason":"Expired","code":410}}
```

- The answer is `410` with reason `Expired`: the history you asked for is no longer kept. A
  Kubernetes client handles it by listing everything again and watching from there.
- In the message, `<BEFORE>` is your `$BEFORE`, and `<N>` is one less than the oldest revision the
  API server still holds.
- This `410` came from the API server, not from etcd. The API server keeps its own short history of
  recent changes, at least about 75 seconds per resource, and the noise pushed the older changes out.
- If you see a stream of events instead, that history hasn't been pushed out yet. Wait a minute and
  run it again.

---

## 8. Compaction and defrag

```bash
etcdctl endpoint status -w json | jq -c '.[0].Status | {dbSize, dbSizeInUse}'
etcdctl compact $(etcdctl endpoint status -w json | jq '.[0].Status.header.revision') --physical
etcdctl endpoint status -w json | jq -c '.[0].Status | {dbSize, dbSizeInUse}'
```

```text
{"dbSize":14733312,"dbSizeInUse":14733312}
{"dbSize":14745600,"dbSizeInUse":606208}
```

```bash
etcdctl defrag
etcdctl endpoint status -w json | jq -c '.[0].Status | {dbSize, dbSizeInUse}'
```

```text
Finished defragmenting etcd member[https://127.0.0.1:2379]. took 11.557333ms
{"dbSize":548864,"dbSizeInUse":540672}
```

- `dbSize` is the size of etcd's file on disk. `dbSizeInUse` is the part of it that holds live data.
- Between the two readings, `compact` also prints `compacted revision` and the revision.
- Compaction dropped the live data from 14.7 MB to 0.6 MB, and the file stayed at 14.7 MB. The freed
  space is still inside the file.
- **Defrag** rewrote the file without that free space, and it shrank to 0.55 MB. On a small file it
  takes milliseconds. On a database of several gigabytes it takes seconds, and etcd answers nothing
  while it runs.
- `--physical` makes `compact` wait until the space is really freed, so the next reading is stable.
- Your sizes depend on how long the noisy producer has been running.

Stop the noise.

```bash
kubectl delete -f noisy-producer.yaml
```

---

## Optional: watch a defrag block

Twelve milliseconds is too short to see anything wait, so make the file bigger first.

```bash
head -c 786432 /dev/urandom | base64 | tr -d '\n' > blob.txt
for i in $(seq 1 300); do printf '%s' "$i" | cat - blob.txt | etcdctl put /demo-big/blob >/dev/null; done
etcdctl endpoint status -w json | jq -c '.[0].Status | {dbSize, dbSizeInUse}'
```

```text
{"dbSize":365551616,"dbSizeInUse":318529536}
```

- `blob.txt` is a 1 MiB value. Each `put` reads it from stdin, because a value that size is too long
  for a command-line argument on macOS.
- All 300 versions of the one key are kept, because nothing has compacted them. That makes a 366 MB
  file.

Time one write on its own, then one that starts 0.2 seconds into a defrag.

```bash
time etcdctl put /not-kubernetes/tick 1
(sleep 0.2; time etcdctl put /not-kubernetes/tick 2) & etcdctl defrag; wait
```

```text
OK
  0.01s user 0.01s system 74% cpu 0.017 total
OK
Finished defragmenting etcd member[https://127.0.0.1:2379]. took 417.990334ms
  0.01s user 0.01s system 7% cpu 0.216 total
```

- On its own, the write took 17 ms.
- During the defrag it took 216 ms, which is the rest of the 418 ms defrag. It waited for the defrag
  to finish.
- zsh's `time` prints the timing line. In an interactive shell it also prints the command in front
  of it.

---

## Clean up

```bash
kind delete cluster --name rv-demo --kubeconfig ./rv-demo.kubeconfig
```

That deletes the cluster. Your usual kubeconfig was never touched, so there is nothing else to undo.

---

## What you saw

- A `resourceVersion` is etcd's revision: one counter for the whole database, moved by every write.
- An object's `resourceVersion` is its key's `mod_revision`, not a count of its own writes.
- A `409` is a write based on an old copy. A `410` is a request for history that is no longer kept.
- Compaction throws old versions away. Only defrag shrinks the file, and etcd waits while it runs.

## Read more

- [KEP-5504: Comparable Resource Version](https://www.kubernetes.dev/resources/keps/5504/), which made
  resourceVersions officially comparable in Kubernetes 1.35
- [Kubernetes API concepts: resource versions](https://kubernetes.io/docs/reference/using-api/api-concepts/#resource-versions)
- [etcd data model](https://etcd.io/docs/v3.6/learning/data_model/)
- [etcd maintenance: compaction and defragmentation](https://etcd.io/docs/v3.6/op-guide/maintenance/)
