# Preparation

in this step, you must configure all hosts to be able to communicate with ssh without needing to use a password. And, the user you are using must be granted sudo access.

## Mapping Host IP
edit the /etc/hosts section so that all nodes can communicate using the hostname.

**Run on all node**
### Change hostname controller and compute node
```
$ hostname controller<number>
$ echo 'controller<number>' > /etc/hostname
```
### Map host on /etc/hosts
```
nano /etc/hosts
...
10.XX.XX.10 controller<number>
10.XX.XX.20 compute1
10.XX.XX.30 compute2
...
```
## Generate SSH key and assign id_rsa.pub to all nodes
**Run only on controller node**
### Create ssh key
```
ssh-keygen -t rsa
```
### Copy controller public key to all nodes
```
ssh-copy-id -i ~/.ssh/id_rsa.pub student@controller<number>
ssh-copy-id -i ~/.ssh/id_rsa.pub student@compute1
ssh-copy-id -i ~/.ssh/id_rsa.pub student@compute2
```
## Creating Sudo User 
**Run all node**

### Register user to sudo group
```
usermod -aG sudo <username>
```
### Enable sudo without password
```
echo "<username> ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/<username>
```
