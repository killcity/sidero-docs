

# Talos/Sidero Cheat Sheet

## Talos

### Configure an endpoint

`talosctl config endpoint <ip>`

---

### Show logs for a service

`talosctl logs networkd --talosconfig <talosconfig> --nodes <ip>`


## Sidero

### Accepting a `server`
1. Find server in pool

`kubectl get servers`

2. Accept server (we can also auto-accept)

```
kubectl patch server <server uid> --type='json' -p='[{"o
p": "replace", "path": "/spec/accepted", "value": false}]'
```
---
### Removing a `machine` from a cluster (and returning it to the pool). 

*Note: This will also remove the node from whatever cluster it's part of.*

Pause `machinedeployment` so it cannot allocate machines.

1. Pick appropriate `machinedeployment`

`kubectl get machinedeployment -o wide`

2. Pause it. This will stop the node from being re-allocated when you remove it.

`kubectl patch machinedeployment <machinedeployment> --type='json' -p='[{ "op": "replace", "path": "/spec/paused", "value": true}]'`

3. Remove machine. This will remove the 'machine' and 'server' from respective clusters.

`kubectl delete machine <machine>`

4. Unpause the `machinedeployment`.

---
### Generate a `kubeconfig` for a new cluster



---
### Generate a `talosconfig` for a new cluster

---
### Scaling a cluster

#### Scale Workers
```
kubectl scale machinedeployment sidero-test-1-workers --replicas=x
```  

#### Scale ControlPlane

```
kubectl scale taloscontrolplane sidero-test-1-cp --replicas=x
```
