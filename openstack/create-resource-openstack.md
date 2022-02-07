### 1. Create external network and external subnet 
```
openstack network create \
--external \
--share \
--provider-network-type flat \
--provider-physical-network physnet1 \
external-network

openstack  subnet create \
--subnet-range 10.50.50.0/24 \
--no-dhcp \
--gateway 10.50.50.1 \
--dns-nameserver 10.50.50.1 \
--allocation-pool start=10.50.50.11,end=10.50.50.100 \
--network external-network \
external-subnet
```
### 2. Create internal network and internal subnet 
```
openstack  network create \
internal-network

openstack subnet create \
--subnet-range 192.168.1.0/24 \
--network internal-network \
internal-subnet
```
### 3. Create pod network and pod subnet
```
openstack network create pod

openstack subnet create --network pod --no-dhcp \
--gateway 10.1.255.254 \
--subnet-range 10.1.0.0/16 \
pod_subnet
```
### 4. Create service network and service subnet, We reserve the first half of the subnet range for the VIPs and the second half for the loadbalancer vrrp ports
```
openstack network create services

openstack subnet create --network services --no-dhcp \
--gateway 10.2.255.254 \
--ip-version 4 \
--allocation-pool start=10.2.128.1,end=10.2.255.253 \
--subnet-range 10.2.0.0/16 \
service_subnet
```
### 5. Create router
```
openstack  router create router1
```
### 6. Create router ports in the internal, pod and service subnets
```
openstack port create --network internal-network --fixed-ip ip-address=192.168.1.1 internal_subnet_router

openstack port create --network pod --fixed-ip ip-address=10.1.255.254 pod_subnet_router

openstack port create --network services --fixed-ip ip-address=10.2.255.254 service_subnet_router
```
### 7. Set external gateway for router
```
openstack  router set \
--external-gateway external-network \
router1
```
### 8. Add the router to the internal, service and the pod subnets
```
openstack router add port router1 internal_subnet_router

openstack router add port router1 pod_subnet_router

openstack router add port router1 service_subnet_router
```
### 9. Create new project and user 
```
openstack project create kuryr

openstack user create --project kuryr --password-prompt support

openstack role add --project kuryr --user support admin

nano /etc/kolla/support-openrc.sh
...
for key in $( set | awk '{FS="="}  /^OS_/ {print $1}' ); do unset $key ; done
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=service
export OS_TENANT_NAME=service
export OS_USERNAME=kuryr
export OS_PASSWORD=123
export OS_AUTH_URL=http://10.20.30.100:35357/v3
export OS_INTERFACE=internal
export OS_ENDPOINT_TYPE=internalURL
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME=RegionOne
export OS_AUTH_PLUGIN=password
...

source /etc/kolla/support-openrc.sh
```
### 10. Create keypair 
```
openstack keypair create --public-key .ssh/id_rsa.pub key-nya
```
### 11. Create image ubuntu 18.04
```
wget https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img

openstack image create --container-format bare --disk-format raw --file bionic-server-cloudimg-amd64.img ubuntu1804
```
### 12. Create flavor for master and worker instance
```
openstack flavor create --disk 15 --ram 4086 --vcpus 2 medium
```
### 13. 


