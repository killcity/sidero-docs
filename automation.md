# Automating Kubernetes Cluster Deployments with Sidero


### DHCP records

Nodes will need to be added to dhcpd.conf. Depending on number of interfaces, it will either have one or two entries (to match with # of nics), similar to how stacki does it. In fact, we can “steal” the stacki configs for every node we want to add and automate modification and reload of the dhcp services every time a machine is to be added. The ips can be pulled from IPAM. `dhcpd` will need to be restarted each time `dhcpd.conf` is modified unless we can find a daemon that performs dynamic reloads. **Also, if we dont want to use fixed-addresses, we can choose to pull from a scope/pool and set hostnames via DDNS updates.
**
```
        host srv-01-14-409.build.eth0 {
            option host-name        "srv-01-14-409";
            hardware ethernet       a0:36:9f:7a:1f:e0;
            fixed-address           10.16.171.5;
        }
  
        host srv-01-14-409.build.eth1 {
            option host-name        "srv-01-14-409";
            hardware ethernet       a0:36:9f:7a:1f:e2;
            fixed-address           10.16.171.5;
        }
```

### Cluster Manifests

We need a command that can take the following args or a tf provider?


#### Example
```
Usage:

create-cluster -n name -p pod-cidr -s service-cidr -e endpoint_ip

Example:

create-cluster -n mycluster -p 10.17.16.0/20 -s 10.19.4.0/22 -e 10.16.170.200
```

This will most likely modify a template (jinja?) and apply...

We also need to set `kube-router` annotations after the cluster is up and add labels where necessary.


### IPMI fixes

Currently, there is some issue with accounts created on DRACS by Sidero. The `sidero` user exists, but doesnt work, even though it shows the correct role.

There is a script that we have written that allows a workaround - where you can use whatever user you want. The argument is the uid of the server.

```
#!/bin/sh

kubectl get secret $1-bmc -o json | jq '.data."user" |= ("username" | @base64)' | kubectl apply -f -
kubectl get secret $1-bmc -o json | jq '.data."pass" |= ("password" | @base64)' | kubectl apply -f -
```

### Adding / Removing nodes

We have two scripts, one to accept machine and the other to remove them from active clusters and return them back to the server pool. Auto-accepting a server can be enabled if desired, but we take a cautious approach and have that happen manually.



#### Pausing a machinedeployment
In order to remove a `machine` without it instantly being re-provisioned, we need to pause the `machinedeployment`

```
#!/bin/sh
# paused a machinedeployment so it cannot allocate machines
kubectl patch machinedeployment $1 --type='json' -p='[{ "op": "replace", "path": "/spec/paused", "value": true}]'
```


#### Accepting a server

```
kubectl patch server $1 --type='json' -p='[{"op": "replace", "path": "/spec/accepted", "value": false}]'
```





