# Getting started with Akri

- Akri stands for "a Kubernetes Resource Interface".
- Means 'edge' in Greek
- CNCF Sandbox project

## Prerequites

- A container runtime (Docker)
- A cluster orchestrator (k3d)
- A device manager (udev)

## Setup 

- Collect device information

```sh
# all devices with 'video' capture capability
udevadm info /dev/video* | grep -e ID_V4L_CAPABILITIES -e DEVNAME -e ID_V4L_PRODUCT

# all info about a particular device (by product name)
udevadm info --export-db --property-match=ID_V4L_PRODUCT="HD Pro Webcam C920" | less
```

- Create a cluster

```sh
k3d cluster create akri
```

- Install Akri with udev

```sh
# get Akri's helm repo
helm repo add akri-helm-charts https://project-akri.github.io/akri/

# install Akri
helm install akri akri-helm-charts/akri \
  --namespace akri --create-namespace \
  --set udev.discovery.enabled=true \
  --set udev.configuration.enabled=true \
  --set udev.configuration.name=akri-udev-video \
  --set udev.configuration.discoveryDetails.udevRules\[0\]='KERNEL=="video[4-6]*", ENV{ID_V4L_CAPABILITIES}==":capture:"' \
  --set udev.configuration.brokerPod.image.repository=ghcr.io/project-akri/akri/udev-video-broker
```

# Deploy a Streaming Application

```sh
# switch to default namespace
kubens default

# deploy the application
kubectl create -f streaming-app.yml

# get service port
kubectl get service/akri-video-streaming-app --output=jsonpath='{.spec.ports[?(@.name=="http")].nodePort}' && echo

# (optional) if the cluster is not using host network
kubectl port-forward svc/akri-video-streaming-app 8080:80
```

## Resources

- https://docs.akri.sh/
- https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/
