
# Building and Deploying Clusters with Sidero

## Prerequisites

You need these tools installed before you can create or manage clusters. In this case, we are building on bare metal so we'll need DHCP and DNS services deployed.

### Tools
```
clusterctl
talosctl
kubectl
```
---
### DHCP and DNS

#### DHCP

Nodes will need to be added to `dhcpd.conf`. Depending on number of interfaces, it will either have one or two entries (to match with # of nics), similar to how stacki does it. In fact, we can "steal" the stacki configs for every node we want to add and automate modification and reload of the dhcp services every time a machine is to be added. The ips can be pulled from IPAM.

*Locating hardware and mapping hostnames*

 We need to make sure we map mac address to hostnames, especially when the location of your servers are embedded in the hostname. If we lose a node in a future k8s cluster, deployed by sidero, we'll need to be able to locate it. This can be done using `labels`, but in the event of a catastrophic failure (lose the sidero or k8s apis), it will help that we have a record of location in our `dhcpd.conf`. Also, we dont necessarily need to set the ips in this file. We could use a scope, lease them out and use DDNS to update PDNS.

* Testing will need to happen in order for two DHCP servers to support the fixed addresses. In the event one of them goes down, the other still still offer leases.
* Naming conventions. The server naming convention used in the example below reflects [type]-[blade location]-[rank]-[rack]. We've found this works really well. We will typically advertise node's duties to `consul` if we want to know what's running on it.

#### Example DHCP entries

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

#### DNS

Along with being added to DHCP, every node will need to either have it's IP modified (if it was previously used) or added to PDNS. The API call is the same regardless. If it already exists, it will be overwritten. If it doesn't, it will be created.

#### Example PDNS API Calls

* *Note: We are making use of consul tags. The calls are specifically targeting the `master` dns (read/write authoritative) node.*
```
# set env variables
export FQDN=`hostname`
export IP=`hostname -i`
export REV_IP=`hostname -i|awk -F'.' '{print $4,$3}' OFS='.'`
export FWD_ZONE=`hostname -d`
export REV_ZONE="16.10.in-addr.arpa"
export APIKEY="redacted"
export REV_NAME=$REV_IP.$REV_ZONE
export DC="iad1"


# add Forward 'A' record via PowerDNS API
echo "adding A record in pdns"
/usr/bin/curl -X PATCH --data '{"rrsets": [ {"name": "'$FQDN.'", "type": "A", "ttl": 60, "changetype": "REPLACE", "records": [ {"content": "'$IP'", "disabled":false } ] } ] }' -H "X-API-Key: $APIKEY" http://master.dns.service.$DC.consul:8081/api/v1/servers/localhost/zones/$FWD_ZONE.

# add Reverse 'PTR' record via PowerDNS API
echo "adding PTR record in pdns"
/usr/bin/curl -X PATCH --data '{"rrsets": [ {"name": "'$REV_NAME.'", "type": "PTR", "ttl": 60, "changetype": "REPLACE", "records": [ {"content": "'$FQDN.'","disabled": false } ] } ] }' -H "X-API-Key: $APIKEY" http://master.dns.service.$DC.consul:8081/api/v1/servers/localhost/zones/$REV_ZONE.
```
---

# Deploy a Bootstrap Cluster

This will deploy a single node bootstrap Sidero control plane running in Kubernetes.

### Talos Install
Make sure you are booting a node onto a **network with dhcp**. Once the box is up, run the following command. Use an iso, ova etc to spin the node up.

```
talosctl apply-config -n <talos node ip> --interactive --insecure
```

This will launch an interactive installation UI in your terminal. 

*Note: You can click between tabs using your mouse.*

Once youve created the single node cluster, it will drop a *talosconfig* file in `~/.talos/config`. This file won't work alone in order to talk to the talos api. **You'll need to set an `endpoint`**, which is either an ip of a *talos node* or the *vip* for a cluster you might've just built.

#### Configure an Endpoint

`talosctl config endpoint <ip>`

*Note: Once you've created an endpoint, you can `cat ~/.talos/config` and notice `endpoint:` now has a value.*

#### Generate `kubeconfig`

`talosctl --talosconfig=/path/to/.talos/config kubeconfig`

Once you've generated a `kubeconfig`, try running a command such as:

`kubectl get node`

You should see the single talos node listed.

### Sidero Install

Now that the cluster is up, we can install Sidero.

Sidero has a [bunch of installation options](https://www.sidero.dev/docs/v0.3/getting-started/installation/), which can be exported as environment vars. For a typical installation, we might use the following. 

*Note: The API endpoint is again, the ip of your single talos node.* We are also explicitly setting the Sidero version in the `clusterctl` command using `-i`.

```
#!/bin/sh
SIDERO_METADATA_SERVER_HOST_NETWORK=true
SIDERO_METADATA_SERVER_PORT=9091
SIDERO_CONTROLLER_MANAGER_HOST_NETWORK=true
SIDERO_CONTROLLER_MANAGER_API_ENDPOINT=<ip of talos node>


clusterctl init --kubeconfig=/path/to/.kube/config -i sidero:v0.3.0-alpha
.0 -b talos -c talos
```

#### Allow pods to run on the control plane

`kubectl taint node sidero-mgmt-test node-role.kubernetes.io/master:NoSchedule-`


---
# Deploying an HA Kubernetes Cluster

## Creating a Cluster Manifest
The cluster manifest will desribe the type of cluster you want to build. How many control plane nodes, how many workers, the type of workers and control plane nodes, etc. We've created a manifest that can be templatized. The values that need to be changed, per-cluster are:


#### cidrBlocks
In this example, we are setting the pod and service `cidrBlocks`.

```
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 10.17.16.0/20
    services:
      cidrBlocks:
      - 10.19.4.0/22
```

Next, we want to set the `controlPlaneEndpoint` ip to the *vip* we setup when we ran the cluster creation script.

#### controlPlaneEndpoint

```
spec:
  controlPlaneEndpoint:
    host: 10.16.170.175
    port: 6443
```


Now, for patches, we need to set the url to the custom kube-router manifest. This manifest will have the tweaks listed below. Next, we need to disable `kube-proxy` as it can interfere with `kube-router`. Also, we need to set the *vip endpoint* to be created on any of the control plane nodes when they spin up.

#### patches

```
spec:
  controlPlaneConfig:
    controlplane:
      generateType: controlplane
      talosVersion: v0.10.2
      configPatches:
        - op: replace
          path: /cluster/network/cni
          value:
            name: custom
            urls:
            - http://mywebserver/sidero-test-1.yaml
        - op: add
          path: /cluster/proxy
          value:
            disabled: true
        - op: add
          path: /machine/network
          value:
            interfaces:
              - interface: eth0
                dhcp: true
                vip:
                  ip: 10.16.170.175
```

### kube-router

We need to package a custom manifest that includes the pod cidr block for the target cluster as well as the endpoint ip. We use kube-router due to it's use of bgp. We customise it not to use any overlay networking or anything that might slow things down.

#### Customising kube-router per-cluster

We are customising most everything in this manifest in the `configMap`. The `kubeconfig` file `kube-router` will be using is rendered...

Pay attention to:

`server: https://10.16.170.170:6443`
`clusterCIDR: 10.17.16.0/20`

Again, `server` should equal the ip of your *vip* and `clusterCIDR` should equal the podCIDR block.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubeconfig
  namespace: kube-system
  labels:
    tier: node
data:
  kubeconfig: |
    apiVersion: v1
    kind: Config
    clusterCIDR: 10.17.16.0/20
    clusters:
    - name: cluster
      cluster:
        certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        server: https://10.16.170.170:6443
    users:
    - name: kube-router
      user:
        tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    contexts:
    - context:
        cluster: cluster
        user: kube-router
      name: kube-router-context
    current-context: kube-router-context
```

#### Entire kube-router Manifest

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-router-cfg
  namespace: kube-system
  labels:
    tier: node
    k8s-app: kube-router
data:
  cni-conf.json: |
    {
       "cniVersion":"0.3.0",
       "name":"mynet",
       "plugins":[
          {
             "name":"kubernetes",
             "type":"bridge",
             "bridge":"kube-bridge",
             "isDefaultGateway":true,
             "ipam":{
                "type":"host-local"
             }
          }
       ]
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-router
  namespace: kube-system
  labels:
    k8s-app: kube-router
spec:
  selector:
    matchLabels:
      k8s-app: kube-router
  template:
    metadata:
      labels:
        k8s-app: kube-router
    spec:
      priorityClassName: system-node-critical
      serviceAccountName: kube-router
      containers:
      - name: kube-router
        image: docker.io/cloudnativelabs/kube-router
        args:
          - '--run-router=True'
          - '--run-firewall=True'
          - '--run-service-proxy=True'
          - '--kubeconfig=/var/lib/kube-router/kubeconfig'
          - '--advertise-cluster-ip'
          - '--advertise-external-ip'
          - '--advertise-loadbalancer-ip'
          - '--nodes-full-mesh=false'
          - '--enable-overlay=false'
          - '--enable-pod-egress=false'
        securityContext:
          privileged: true
        imagePullPolicy: Always
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: KUBE_ROUTER_CNI_CONF_FILE
          value: /etc/cni/net.d/10-kuberouter.conflist
        livenessProbe:
          httpGet:
            path: /healthz
            port: 20244
          initialDelaySeconds: 10
          periodSeconds: 3
        volumeMounts:
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true
        - name: cni-conf-dir
          mountPath: /etc/cni/net.d
        - name: kubeconfig
          mountPath: /var/lib/kube-router
          readOnly: true
        - name: xtables-lock
          mountPath: /run/xtables.lock
          readOnly: false
      initContainers:
      - name: install-cni
        image: docker.io/cloudnativelabs/kube-router
        imagePullPolicy: Always
        command:
        - /bin/sh
        - -c
        - set -e -x;
          if [ ! -f /etc/cni/net.d/10-kuberouter.conflist ]; then
            if [ -f /etc/cni/net.d/*.conf ]; then
              rm -f /etc/cni/net.d/*.conf;
            fi;
            TMP=/etc/cni/net.d/.tmp-kuberouter-cfg;
            cp /etc/kube-router/cni-conf.json ${TMP};
            mv ${TMP} /etc/cni/net.d/10-kuberouter.conflist;
          fi
        volumeMounts:
        - name: cni-conf-dir
          mountPath: /etc/cni/net.d
        - name: kube-router-cfg
          mountPath: /etc/kube-router
      - name: install-cni-talos
        image: ghcr.io/talos-systems/install-cni:v0.3.0
        imagePullPolicy: IfNotPresent
        command:
        - /install-cni.sh
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /host/opt/cni/bin/
          name: host-cni-bin
      hostNetwork: true
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoExecute
        operator: Exists
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: cni-conf-dir
        hostPath:
          path: /etc/cni/net.d
      - name: kube-router-cfg
        configMap:
          name: kube-router-cfg
      - name: kubeconfig
        configMap:
          name: kubeconfig
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
      - name: host-cni-bin
        hostPath:
          path: /opt/cni/bin
          type: ""
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-router
  namespace: kube-system

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kube-router
  namespace: kube-system
rules:
  - apiGroups:
    - ""
    resources:
      - namespaces
      - pods
      - services
      - nodes
      - endpoints
    verbs:
      - list
      - get
      - watch
  - apiGroups:
    - "networking.k8s.io"
    resources:
      - networkpolicies
    verbs:
      - list
      - get
      - watch
  - apiGroups:
    - extensions
    resources:
      - networkpolicies
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kube-router
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-router
subjects:
- kind: ServiceAccount
  name: kube-router
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubeconfig
  namespace: kube-system
  labels:
    tier: node
data:
  kubeconfig: |
    apiVersion: v1
    kind: Config
    clusterCIDR: 10.17.16.0/20
    clusters:
    - name: cluster
      cluster:
        certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        server: https://10.16.170.170:6443
    users:
    - name: kube-router
      user:
        tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    contexts:
    - context:
        cluster: cluster
        user: kube-router
      name: kube-router-context
    current-context: kube-router-context
```    

### Environment 

The environment allows you to tweak bootloader settings.

#### Example

```
apiVersion: metal.sidero.dev/v1alpha1
kind: Environment
metadata:
  name: default
spec:
  kernel:
    url: "https://github.com/talos-systems/talos/releases/download/v0.10.2/vmlin
uz-amd64"
    sha512: ""
    args:
      - init_on_alloc=1
      - init_on_free=1
      - slab_nomerge
      - pti=on
      - consoleblank=0
      - random.trust_cpu=on
      - ima_template=ima-ng
      - ima_appraise=fix
      - ima_hash=sha512
      - console=tty0
      - console=ttyS1,115200n8
      - earlyprintk=ttyS1,115200n8
      - panic=0
      - printk.devkmsg=on
      - talos.platform=metal
      - talos.config=http://10.16.171.1:9091/configdata?uuid=
  initrd:
    url: "https://github.com/talos-systems/talos/releases/download/v0.10.2/initr
amfs-amd64.xz"
    sha512: ""
```
    
### ServerClass

ServerClasses allow you to extract the nodes personality and organize.

#### Example - Dell Worker
*Note: Notice how we are also setting block device and defining bonding.*

```
apiVersion: metal.sidero.dev/v1alpha1
kind: ServerClass
metadata:
  name: bm-worker
spec:
  qualifiers:
    systemInformation:
      - manufacturer: Dell Inc.
  configPatches:
    - op: replace
      path: /machine/install/disk
      value: /dev/sda
    - op: add
      path:   /machine/network
      value:
       interfaces:
         - interface: bond0
           dhcp: true
           bond:
           mode: 802.3ad
           lacprate: fast
           hashpolicy: layer2
           miimon: 100
           updelay: 200
           downdelay: 200
           interfaces:
             - eth4
             - eth5
```

#### Example - VM Control Plane
*Note: Notice how we are also setting block device and setting shared endpoint ip.*

```
apiVersion: metal.sidero.dev/v1alpha1
kind: ServerClass
metadata:
  name: vm-cp
spec:
  qualifiers:
    systemInformation:
      - manufacturer: VMware, Inc.
  configPatches:
    - op: replace
      path: /machine/install/disk
      value: /dev/sda
    - op: add
      path:   /machine/network
      value:
       interfaces:
         - interface: eth0
           dhcp: true
           vip:
             ip: 10.16.170.160
```


---

## Apply Manifest


## Generate Talosconfig for new cluster

```
#!/bin/sh

kubectl get talosconfig -l cluster.x-k8s.io/cluster-name=$1 -o yaml -o jsonpath='{.items[0].status.talosConfig}' > $1-talosconfig.yaml
```

## Generate Kubeconfig for new cluster

Run this against any of the cp nodes
```
# talosctl config endpoint 10.16.170.175 --talosconfig <mytalosconfig>
```
if this doesnt work, manually add endpoint in your fresh `talosconfig`, example below.
```
contexts:
  sidero-test-1:
    endpoints:
        - 10.16.170.175
```
```
# talosctl kubeconfig --talosconfig=<mytalosconfig> --nodes 10.16.171.52
```






