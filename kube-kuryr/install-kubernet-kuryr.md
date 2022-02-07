# Install kubernetes cluster and kuryr

### 1. Update and upgrade paket
```
$ sudo apt update -y && sudo apt upgrade -y
```
### 2. Set time-zone
```
$ sudo timedatectl set-timezone Asia/Jakarta
```
### 3. Load kernel module 
```
$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
.bridge.bridge-nf-call-ip6tables = 1
.bridge.bridge-nf-call-iptables = 1
EOF

$ sudo sysctl --system
$ sudo modprobe br_netfilter
```
### 4. Install docker 
```
sudo apt install docker.io
```
### 5. Configure docker storage driver
Copy this configuration to daemon.json
```
sudo nano /etc/docker/daemon.json
...
{
    "exec-opts": ["native.cgroupdriver=systemd"],
	"log-driver": "json-file",
	"log-opts": {
	"max-size": "100m"
	},
	"storage-driver": "overlay2"
}
...
```
```
$ sudo systemctl restart docker
```
### 6. Add repository kubernetes
```
$ sudo apt-get install -y apt-transport-https curl

$ sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

$ cat Instalasi paket kubernetes<<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

$ sudo apt update -y
```
### 7. Install packet kubernetes 
```
$ sudo apt-get install kubelet kubeadm kubectl

$ sudo apt-mark hold kubelet kubeadm kubectl
$ sudo systemctl enable --now kubelet
```
### 8. Install and configure haproxy and keepalived
Execute on master 1-3
```
$ sudo apt install haproxy keepalived -y

$ sudo nano /etc/haproxy/haproxy.cfg
...
frontend apiservers
        bind *:8443
        mode tcp
        option tcplog
        default_backend k8s_apiservers

backend k8s_apiservers
        mode tcp
        option tcplog
        option tcp-check
        option log-health-checks
        balance roundrobin
        default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
        server cluster-01 k8s-master1:6443 check
        server cluster-02 k8s-master2:6443 check
        server cluster-03 k8s-master3:6443 check
...
$ sudo systemctl restart haproxy
```
#### Execute on master1
```
$ sudo nano /etc/keepalived/keepalived.conf
...
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance LB_VIP {
    interface ens3
    state MASTER
    priority 101
    virtual_router_id 52

    authentication {
        auth_type PASS
        auth_pass 123
    }

    virtual_ipaddress {
        192.168.1.100
    }

    track_script {
        chk_haproxy
    }
}
...

$ systemctl enable keepalived
$ systemctl restart keepalived
```
#### Execute on master2
```
$ sudo nano /etc/keepalived/keepalived.conf
...
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance LB_VIP {
    interface ens3
    state BACKUP
    priority 100
    virtual_router_id 52

    authentication {
        auth_type PASS
        auth_pass 123
    }

    virtual_ipaddress {
        192.168.1.100
    }

    track_script {
        chk_haproxy
    }
}
...

$ systemctl enable keepalived
$ systemctl restart keepalived
```
#### Execute on master3
```
$ sudo nano /etc/keepalived/keepalived.conf
...
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance LB_VIP {
    interface ens3
    state BACKUP
    priority 99
    virtual_router_id 52

    authentication {
        auth_type PASS
        auth_pass 123
    }

    virtual_ipaddress {
        192.168.1.100
    }

    track_script {
        chk_haproxy
    }
}
...

$ systemctl enable keepalived
$ systemctl restart keepalived
```
### 9. Init kubernetes cluster
Execute on master1
```
kubeadm init --kubernetes-version 1.23.3 --service-cidr 10.2.0.0/17 --pod-network-cidr 10.1.0.0/16 \
--upload-certs --control-plane-endpoint "k8s-vrrp:8443" \
--apiserver-cert-extra-sans=k8s-vrrp,10.2.255.254,192.168.1.100,172.50.50.100
```
**Note! copy command for join kube master and worker**
### 10. Join kube master 2 and 3
```
$ sudo kubeadm join k8s-vrrp:8443 \
    --token <token> \
    --discovery-token-ca-cert-hash <ca-cert-hash> \
    --control-plane --certificate-key <cert-key>
```
### 11. Join kube worker (but after kuryr deployment applyed)
```
sudo kubeadm join k8s-vrrp:8443 \
    --token <token> \
    --discovery-token-ca-cert-hash <ca-cert-hash>
```
### 12. Label node kubernetes worker
```
$ kubectl label nodes <worker-hostname> node-role.kubernetes.io/worker=
```
## Install CNI Kuryr

### 1. Create image kuryr controller and kuryr cni or you're can use a image I have made on docker hub
```
$ git clone http://git.openstack.org/openstack/kuryr-kubernetes -b stable/ussuri
$ cd kuryr-kubernetes
$ docker build -t kuryr/controller -f controller.Dockerfile .
$ docker build -t kuryr/cni -f cni.Dockerfile .
```
**pull image kuryr on my docker hub**
```
docker pull mstock14/kuryr:cni-u
docker pull mstock14/kuryr:controller-u
```
### Copy image kuryr controller and cni to node master and worker 
### 2. Create configuration kuryr controller
```
nano ~/kuryr.conf
...
[DEFAULT]
use_stderr = true
lock_path = /var/kuryr-lock
bindir = /usr/local/libexec/kuryr

[binding]
default_driver = kuryr.lib.binding.drivers.vlan
link_iface = ens3
[cache_defaults]

[cni_daemon]
docker_mode = true
netns_proc_dir = /host_proc

[cni_health_server]
[health_server]

[kubernetes]
api_root = https://k8s-vrrp:8443
pod_vif_driver = nested-vlan

[kuryr-kubernetes]
[namespace_subnet]

[neutron]
username = admin
user_domain_name = Default
password = G4YfG0uowFO90MAKg2wwD318uhPRF9uMBWbsabHV
project_name = admin
project_domain_name = Default
auth_type = password
auth_url = http://10.20.30.100:5000
#cafile = /etc/ssl/certs/kuryr-ca-bundle.crt
insecure=true

[neutron_defaults]
ovs_bridge = br-int
external_svc_net = dd96e891-3b16-429d-ac91-da5661070976
external_svc_subnet = 150dce25-041b-452b-8037-5c4e093543dc
pod_security_groups = ad15a556-7436-4a68-8637-7c6309df8be7
service_subnet = c01322ce-a913-4174-8b6d-8f238e048dc4
pod_subnet = 6db554e7-c678-4f4a-97f8-81b224e7c8a3
project = 27141c66bdec47ebb569a81f541ceb06

[node_driver_caching]
[octavia_defaults]

[pod_vif_nested]
worker_nodes_subnet = 3699c1cf-a388-4d04-8002-45ba3300ba85

[pool_manager]
[sriov]
[subnet_caching]
[vif_pool]

[vif_plug_ovs_privileged]
helper_command = privsep-helper

[vif_plug_linux_bridge_privileged]
helper_command = privsep-helper
...
```
### 3. Generate config kuryr
```
$ mkdir ~/kuryr-generated-config
$ ./tools/generate_k8s_resource_definitions.sh ~/kuryr-generated-config ~/kuryr.conf
```
### 4. Apply kuryr kubernetes crd
```
$ cd kuryr-kubernetes/kubernetes_crds/
$ kubectl apply -f network_attachment_definition_crd.yaml
$ kubectl apply -f kuryr_crds
```
### 5. Aplly config kuryr 
```
$ cd kuryr-generated-config/
$ kubectl apply -f config_map.yml -n kube-system
$ kubectl apply -f certificates_secret.yml -n kube-system
$ kubectl apply -f service_account.yml -n kube-system
```
```
$ nano cni_ds.yml 
...
 readinessProbe:
          tcpSocket:
            port: 8090
          initialDelaySeconds: 60
          timeoutSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 8090
          initialDelaySeconds: 60
...
$ nano controller_deployment.yml 
...
        readinessProbe:
          tcpSocket:
            port: 8082
          timeoutSeconds: 5
        livenessProbe:
          tcpSocket:
            port: 8082
          initialDelaySeconds: 15
...

$ kubectl apply -f controller_deployment.yml -n kube-system
$ kubectl apply -f cni_ds.yml -n kube-system
```

### 6. On worker node make sure image kuryr available
**pull image kuryr on my docker hub**
```
docker pull mstock14/kuryr:cni-u
docker pull mstock14/kuryr:controller-u
```
### 7. Join worker to master

## Kubernetes Post Deploy
### 1. Create openstack loadbalancer for kubernetes api (execute on controller1)
```
$ openstack loadbalancer create --vip-address 10.2.0.1 --vip-subnet-id service_subnet --name default/kubernetes

$ openstack loadbalancer pool create --name default/kubernetes:HTTPS:443 --protocol HTTPS --lb-algorithm LEAST_CONNECTIONS --loadbalancer default/kubernetes

$ openstack loadbalancer member create --name k8s-master --address 192.168.1.11 --protocol-port 6443 default/kubernetes:HTTPS:443

$ openstack loadbalancer member create --name k8s-master --address 192.168.1.12 --protocol-port 6443 default/kubernetes:HTTPS:443

$ openstack loadbalancer member create --name k8s-master --address 192.168.1.13 --protocol-port 6443 default/kubernetes:HTTPS:443

$ openstack loadbalancer listener create --name default/kubernetes:HTTPS:443 --protocol HTTPS --default-pool default/kubernetes:HTTPS:443 --protocol-port 443 default/kubernetes
```
### 2. Configure security group allow from lbaas vrrp to kube-api master
Security group name : service_pod_access_sg
- Port: 6443
- Prefix: 10.210.0.0/16
- Protocol: tcp

### 3. Configure PodNodeSelector on kube api (Execute on all master node)
```
$ nano /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --enable-admission-plugins=NodeRestriction,PodNodeSelector
    ...
```
### 4. Configure RemoveSelfLink on kube api (Execute on all master)
```
$ nano /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --feature-gates=RemoveSelfLink=false
    ...
```
### 5. Verification

- On kube master, create pod and then check openstack port list
- On kube master, expose service and then check openstack loadbalancer list



