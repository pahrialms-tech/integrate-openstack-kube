# Kuryr-Kubernetes

The OpenStack Kuryr project enables native Neutron-based networking in Kubernetes. With Kuryr-Kubernetes it's now possible to choose to run both OpenStack VMs and Kubernetes Pods on the same Neutron network if your workloads require it or to use different segments and, for example, route between them.

## Prerequities
I used 7 nodes for OpenStack and Kubernetes cluster, and use 2 network interface each node.

OpenStack Nodes
 - 1 Controller Node
 - 2 Compute Node
 - Ubuntu based
 - OpenStack Ussuri
 - Octavia need to Installed as Load Balancing

Kubernetes Nodes
 - 3 Master Node (with High Avibility)
 - 2 Worker Node
 - Ubuntu based
 - Kubernetes (kubelet, kubectl, kubeadm)
 - Pod Network 10.1.0.0/16
 - Service Network 10.2.0.0/16

## Integration Openstack Kubernet with kuryr Installation

1. [Pre-Deploy Openstack](https://github.com/pahrialms-tech/integrate-openstack-kube/blob/main/openstack/preparation.md) 
2. [Deploy Openstack with octavia](https://github.com/pahrialms-tech/integrate-openstack-kube/blob/main/openstack/install-openstack-with-octavia.md) 
3. [Create Resource Openstack](https://github.com/pahrialms-tech/integrate-openstack-kube/blob/main/openstack/create-resource-openstack.md)
4. [Deploy kubernetes-kuryr](https://github.com/pahrialms-tech/integrate-openstack-kube/blob/main/kube-kuryr/install-kubernet-kuryr.md)
5. [Test](https://github.com/pahrialms-tech/integrate-openstack-kube/blob/main/kube-kuryr/testing.md)
