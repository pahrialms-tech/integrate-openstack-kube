# Verification

### Test deployment
On kube master, create pod and then check openstack port list
```
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
kubectt get pod -w
kubectl get pod -o wide
```
### After pod running, on kube master, expose service and then check openstack loadbalancer list
```
kubectl expose deployment nginx-deployment

openstack loadbalancer list
```
### So if you want pod can access externally, you can give floating ip to loadbalancer
![image](https://user-images.githubusercontent.com/99278834/153572420-617f1524-aae4-45c0-b7de-6f5562240d40.png)

