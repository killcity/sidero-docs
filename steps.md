# Latest Notes

### DNS
    update dhcpd.conf template?
    REST call?
    look at other DNS solutions (Virtual BMC for VSphere?)
    {% for host in subnet.hosts %}
      host {{ host.name }}.{{ host.interface }} {
        hardware ethernet {{ host.mac }};
        fixed-address {{ host.ip }};
      }
    {% endfor %}
        etc...
### cluster YML
    CIDR block data from D42?
### kube-router YML
    stored in /var/www/html on bootstrap server
    http://sidero-mgmt/michaelbay-test.yaml
    Auto-accept? (accept-all.sh)
### Change IPMI
    scripts/change_ipmi.sh
### Generate talosconfig
    get-talosconfig.sh
    talosctl config endpoint 10.16.170.175
### Create kubeconfig
    talosctl kubeconfig --talosconfig=<talosconfig.yaml> --nodes <node>
### Upgrade kube:
    talosctl --talosconfig <talosconfig.yaml> --nodes <single CPl node> upgrade-k8s --from 1.19.4 --to 1.20.1
### Upgrade kubelet:
    talosctl -n < endpoint-IP > patch mc -p '[{"op": "replace", "path": "/machine/kubelet/image", "value": "ghcr.io/talos-systems/kubelet:v1.20.1"}]' --talosconfig michaelbay-test-talosconfig.yaml
    
### Virtual BMC
https://github.com/kurokobo/virtualbmc-for-vsphere/wiki/Containerized-VirtualBMC-for-vSphere

![](https://i.imgur.com/1A1NdWB.png)


