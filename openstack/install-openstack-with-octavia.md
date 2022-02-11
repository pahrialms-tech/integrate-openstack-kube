# Install Openstack Ussuri and Octavia with Kolla-Ansible
After all the [preparations](https://github.com/pahrialms-tech/integrate-openstack-kube/blob/main/openstack/preparation.md) are complete, now we try to install the multi-node OpenStack cluster with Kolla-ansible according to the existing topology.

**Via Kolla-Ansible, Still Difficult**, Setting up Octavia has been an ordeal, and deserves its own page. There are many issues with the default installation not explained in docs, not captured in Ansible, and some misdocumented things, also.So that **you must be carefull !!**.

Run command below **only on the controller node**

### 1. Update repository
```
sudo apt update -y
```
### 2. Install dependencies
```
sudo apt-get install python3-dev libffi-dev gcc libssl-dev python3-selinux python3-setuptools python3-venv -y  
```
### 3. Create a virtual environment and activate it  
```
python3 -m venv kolla-venv  
source kolla-venv/bin/activate  
```
### 4. Install ansible and kolla-ansible  
```
pip install -U pip  
pip install ansible==2.9.13  
pip install kolla-ansible==10.2.0  
```
### 5. Create the /etc/kolla directory  
```
sudo mkdir -p /etc/kolla  
sudo chown $USER:$USER /etc/kolla  
```
### 6. Copy globals.yml and passwords.yml to /etc/kolla directory  
```
cp -r kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla  
```
### 7. Copy all-in-one and multinode inventory files to the current directory  
```
cp kolla-venv/share/kolla-ansible/ansible/inventory/* .  
```
### 8. Configure multinode inventory. [screenshot]  

Make changes to several sections as below  
```
nano ~/multinode  
...
[control]
controller

[network]
controller

[compute]
compute1
compute2

[monitoring]
controller

[storage]
compute1
compute2

[deployment]
localhost ansible_connection=local
...
Do not make changes in other parts
```
### 9. Configure ansible

Tuned ansible configuration environment, add the following options to the Ansible configuration file /etc/ansible/ansible.cfg
```
sudo mkdir -p /etc/ansible
sudo nano /etc/ansible/ansible.cfg
...
[defaults]
host_key_checking=False
pipelining=True
forks=100
...
```
### 10. Test ansible connectivity using ping module. [screenshot]
```
ansible -i multinode all -m ping
```
### 11. Generate kolla password
```
kolla-genpwd
```
### 12. Configure your openstack cluster on kolla globals.yml
```
nano /etc/kolla/globals.yml
Uncomment each line at below on globals.yml
...
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "ussuri"
network_interface: "ens3"
neutron_external_interface: "ens4"
kolla_internal_vip_address: "10.50.50.100"
enable_octavia: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
...
```
confirmation global.yml config
```
cat /etc/kolla/globals.yml | grep -v "#" |  tr -s [:space:]
```

### 13. Create cinder-volumes VG

Create physical volume (PV)
```
sudo pvcreate /dev/vdb 
```
Create volume group (VG)
```
sudo vgcreate cinder-volumes /dev/vdb 
```
Verify VG created.
```
sudo vgs
```
### 14. Generate Certificate for Octavia Amphora  
Use this manual [guide](https://github.com/pahrialms-tech/integrate-openstack-kube/blob/main/certificate/octavia-cert-manual.md) for recomendation generate certificate octavia amphora.  

### 15. Deployment

Bootstrap servers with kolla deploy dependencies:
```
kolla-ansible -i ./multinode bootstrap-servers
```
Do pre-deployment checks for hosts:
```
kolla-ansible -i ./multinode prechecks
```
Finally proceed to actual OpenStack deployment:

this will probably take 10 minutes or more
```
kolla-ansible -i ./multinode deploy
```
Run post-deploy to generate openstack admin-openrc.sh
```
kolla-ansible -i ./multinode post-deploy
```
### 16. Install openstack client and verify openstack cluter

Install openstack client using pip
```
pip install openstackclient
source /etc/kolla/admin-openrc.sh
```

### 17. Setup Octavia Service
Create octavia openrc
```
grep octavia_keystone /etc/kolla/passwords.yml
...
octavia_keystone_password: NTOX7ht7ohPRXiVmb6vckhSc1y1MykvRzvbuocKO

vi /etc/kolla/octavia-openrc.sh
...
# Clear any old environment that may conflict.
for key in $( set | awk '{FS="="}  /^OS_/ {print $1}' ); do unset $key ; done
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=service
export OS_TENANT_NAME=service
export OS_USERNAME=octavia
export OS_PASSWORD=NTOX7ht7ohPRXiVmb6vckhSc1y1MykvRzvbuocKO
export OS_AUTH_URL=http://10.20.30.100:35357/v3
export OS_INTERFACE=internal
export OS_ENDPOINT_TYPE=internalURL
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME=RegionOne
export OS_AUTH_PLUGIN=password
...
```
```
source /etc/kolla/octavia-openrc.sh
```
### 18. Create Amphora Image
```
sudo apt install -y qemu-utils git kpartx debootstrap
git clone https://opendev.org/openstack/octavia -b stable/ussuri

deactivate
python3 -m venv disk-builder
source disk-builder/bin/activate
pip install diskimage-builder

cd octavia/diskimage-create
./diskimage-create.sh
```

### 19. Register the image in Glance
```
deactivate
source ~/kolla-venv/bin/activate
openstack image create amphora-x64-haproxy.qcow2 --container-format bare --disk-format qcow2 --private --tag amphora --file amphora-x64-haproxy.qcow2
```
### 20. Create amphora flavor
```
openstack flavor create --vcpus 1 --ram 1024 --disk 3 "amphora" --private
```

### 21. Create amphora sec group
```
openstack security group create lb-mgmt-sec-grp
openstack security group rule create --protocol icmp lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 22 --ingress lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 9443 --ingress lb-mgmt-sec-grp
openstack security group rule create --protocol udp --dst-port 5555 --egress lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 80 --ingress lb-mgmt-sec-grp
```
### 22. Create amphora Keypair
```
openstack keypair create --public-key ~/.ssh/id_rsa.pub octavia_ssh_key
```
### 23. Copy dhclient.conf to other
```
sudo mkdir -m755 -p /etc/dhcp/octavia
cd ~
sudo cp octavia/etc/dhcp/dhclient.conf /etc/dhcp/octavia
```

### 24. Create Network for Octavia Management
```
OCTAVIA_MGMT_SUBNET=172.16.0.0/16
OCTAVIA_MGMT_SUBNET_START=172.16.0.100
OCTAVIA_MGMT_SUBNET_END=172.16.31.254
OCTAVIA_MGMT_PORT_IP=172.16.0.2

openstack network create lb-mgmt-net
openstack subnet create --subnet-range $OCTAVIA_MGMT_SUBNET --allocation-pool start=$OCTAVIA_MGMT_SUBNET_START,end=$OCTAVIA_MGMT_SUBNET_END --network lb-mgmt-net lb-mgmt-subnet

SUBNET_ID=$(openstack subnet show lb-mgmt-subnet -f value -c id) 
PORT_FIXED_IP="--fixed-ip subnet=$SUBNET_ID,ip-address=$OCTAVIA_MGMT_PORT_IP"

MGMT_PORT_ID=$(openstack port create --security-group lb-mgmt-sec-grp --device-owner Octavia:health-mgr --host=$(hostname) -c id -f value --network lb-mgmt-net $PORT_FIXED_IP octavia-health-manager-listen-port)

MGMT_PORT_MAC=$(openstack port show -c mac_address -f value $MGMT_PORT_ID)

docker exec -it openvswitch_vswitchd ovs-vsctl -- --may-exist add-port br-int o-hm0 -- set Interface o-hm0 type=internal -- set Interface o-hm0 external-ids:iface-status=active -- set Interface o-hm0 external-ids:attached-mac=$MGMT_PORT_MAC -- set Interface o-hm0 external-ids:iface-id=$MGMT_PORT_ID

ip link set dev o-hm0 address $MGMT_PORT_MAC  

dhclient -v o-hm0 -cf /etc/dhcp/octavia/dhclient.conf

ip address
ip link 
docker exec -it openvswitch_vswitchd ovs-vsctl show

sudo ip route delete default via 172.16.0.1 dev o-hm0

NOTE: IF dhcpclient is already running:
ps -eLF | grep dhclient
kill [PID]
```

### 25. Add octavia resource id into globals.yml
```
openstack network show lb-mgmt-net | awk '/ id / {print $4}'
openstack security group show lb-mgmt-sec-grp | awk '/ id / {print $4}'
openstack flavor show amphora | awk '/ id / {print $4}'

vi /etc/kolla/globals.yaml
...
octavia_loadbalancer_topology: "SINGLE" ## Default Topology Capability Octavia
octavia_amp_boot_network_list: "52c2948b-7eb8-4074-b3c6-f15e2b36ce6b" ## ID Network Management Octavia
octavia_amp_secgroup_list: "b9ee14b5-58f1-4bce-9531-465c427c07af" ## ID Sec Group Octavia
octavia_amp_flavor_id: "8b266c5a-d70a-4f45-b1e0-9ad02f7b5ae5" ## ID Default Flavor Octavia
...
```
### 26. Change Octavia Health Manager Config
```
sudo vi /etc/kolla/config/octavia.conf
...
[health_manager]
bind_ip = 172.16.0.2
controller_ip_port_list = 172.16.0.2:5555
```
### 27. Reconfigure Octavia
```
kolla-ansible reconfigure -t octavia
```

