# k8sChw.com

Setup camptonhillsweather.com in a Kubernetes cluster as deployment named chwcom; expose as
service chwcom; connect to the internet as camptonhillsweather.com through an HTTPRoute
pointing to the Envoy Gateway. Get realtime weather data from an NFS mount through the
PV/PVC named chwcom-persistent-storage.

Source image [InstallCHW.com](https://github.com/jkozik/InstallCHW.com)

## Directory structure

```
k8sChw.com/
├── chwcom-pv.yml          # PersistentVolume — NFS 192.168.100.153:/home/nfs/weather-stations/chwcom/public_html
├── chwcom-pvc.yml         # PersistentVolumeClaim (storageClass: nfs-weather, 5Gi ROX)
├── chwcom-deploy.yml      # Deployment (1 replica, jkozik/chw.com:v1.12b)
├── chwcom-svc.yml         # NodePort service
├── chwcom-httproute.yaml  # HTTPRoute — camptonhillsweather.com via Envoy Gateway port 30458
├── README.md              # This file
└── old/
    └── chwcom-ingress.yml # Retired nginx Ingress (kept for reference)
```

## Prerequisites

- Envoy Gateway running with `weather-gateway` on NodePort 30458
- NFS server 192.168.100.153 exporting `/home/nfs/weather-stations/chwcom/public_html`
- `nfs-common` installed on all cluster nodes

## Deploy

Apply in order:

```bash
cd ~/projects/k8sChw.com

# 1. Storage
kubectl apply -f chwcom-pv.yml
kubectl apply -f chwcom-pvc.yml

# Verify PVC is Bound before continuing
kubectl get pv,pvc -l app=chwcom

# 2. Application
kubectl apply -f chwcom-svc.yml
kubectl apply -f chwcom-deploy.yml

# 3. Routing
kubectl apply -f chwcom-httproute.yaml
```

## Verify

```bash
kubectl get deployment,service,pod,httproute,pv,pvc -l app=chwcom

# Expected:
# deployment.apps/chwcom   1/1
# service/chwcom           NodePort  80:<nodeport>/TCP
# pod/chwcom-<hash>        1/1 Running
# httproute/chwcom-route   camptonhillsweather.com, www.camptonhillsweather.com
# pv/chwcom-persistent-storage   Bound
# pvc/chwcom-persistent-storage  Bound
```

Test via NodePort directly:
```bash
curl http://<node-ip>:<nodeport>/ | head -5
```

Test via Envoy Gateway (matches production path):
```bash
curl -H "Host: camptonhillsweather.com" http://<node-ip>:30458/ | head -5
# Should return Campton Hills weather HTML
```

Verify live NFS weather data is visible inside the pod:
```bash
kubectl exec -it deploy/chwcom -- ls /var/www/html/mount
# Expected: cumulus  saratoga  (and other weather data dirs)
```

## Cloudflare tunnel

Point the `camptonhillsweather.com` and `www.camptonhillsweather.com` public hostnames in
the Cloudflare Zero Trust tunnel to:
```
http://<node-ip>:30458
```

## NFS share (reference)

The NFS export is on 192.168.100.153 (dell3):
```
/home/nfs/weather-stations/chwcom/public_html  192.168.100.0/24(ro,sync,no_root_squash)
```

Mounted read-only at `/var/www/html/mount` inside the container.

## Build image / push to Docker Hub

```bash
docker login
docker tag jkozik/chw.com jkozik/chw.com:v1
docker push jkozik/chw.com:v1
```

## Ingress → HTTPRoute migration

Ingress (nginx) has been deprecated on this cluster. Traffic is managed via the Kubernetes
Gateway API implemented by Envoy Gateway. The old ingress yaml is preserved in `old/` for
reference only — do not apply it on the new cluster.
