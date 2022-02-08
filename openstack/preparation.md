# Preparation
### Change hostname controller and compute node
**Run on all node**
```
$ hostname controller<number>
$ echo 'controller<number>' > /etc/hostname
```
### Map host on /etc/host
```
nano /etc/hosts
...
10.XX.XX.10 controller<number>
10.XX.XX.20 compute1
10.XX.XX.30 compute2
...
```
### Create ssh key
**Run only on controller node**
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

### Register user to sudo group
```
usermod -aG sudo <username>
```
### Enable sudo without password
```
echo "<username> ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/<username>
```
