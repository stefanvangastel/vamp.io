---
title: Quick start Kubernetes
type: documentation
weight: 90
menu:
    main:
      parent: installation
---

# Vamp with Kubernetes on Google Container Engine

>**Note**: Kubernetes support is still in Alpha.

An important part of Vamp is that it's container-scheduler agnostic. So this means Vamp also works with Kubernetes. Let's set it up!

## Prerequisites

- Google Container Engine cluster
- Key-value store like ZooKeeper, Consul or etcd
- Elasticsearch and Logstash

>**Note**: [Minikube](https://github.com/kubernetes/minikube) can be also used.

It is advisable to try out the official [Quickstart for Google Container Engine](https://cloud.google.com/container-engine/docs/quickstart) tutorial first.
Simple way to create a new GKE cluster:

- open Google Cloud Shell
- set a zone, e.g. `gcloud config set compute/zone europe-west1-b`
- create a cluster `vamp` using these default parameters: `gcloud container clusters create vamp`

After the (new) Kubernetes cluster is setup, we are going to continue with the installation using Kubernetes CLI `kubectl`.
You can use `kubectl` directly from the Google Cloud Shell, e.g. to check the Kubernetes client and server version:
{{% copyable %}}
```bash
kubectl version
```
{{% /copyable %}}

Let's deploy `etcd` - the installation is based on this [tutorial](https://github.com/coreos/etcd/tree/master/hack/kubernetes-deploy).
Execute: 
{{% copyable %}}
```bash

kubectl create \
        -f https://raw.githubusercontent.com/magneticio/vamp-docker/master/vamp-kubernetes/etcd.yml
```
{{% /copyable %}}

Deploy [Elasticsearch and Logstash](https://github.com/magneticio/elastic) with a proper Vamp Logstash configuration:
{{% copyable %}}
```bash
kubectl run elastic --image=magneticio/elastic:2.2
kubectl expose deployment elastic --protocol=TCP --port=9200 --name=elasticsearch
kubectl expose deployment elastic --protocol=UDP --port=10001 --name=logstash
kubectl expose deployment elastic --protocol=TCP --port=5601 --name=kibana
```
{{% /copyable %}}

>**Note**: This is not a production grade setup. You would also need to take care of persistence and running multiple replicas of each pod.

## Running Vamp

Let's run Vamp gateway agent as a `daemon set` first:
{{% copyable %}}
```bash
kubectl create \
        -f https://raw.githubusercontent.com/magneticio/vamp-docker/master/vamp-kubernetes/vga.yml
```
{{% /copyable %}}

Deploying Vamp:
{{% copyable %}}
```bash
kubectl run vamp --image=magneticio/vamp:0.9.0-kubernetes
kubectl expose deployment vamp --protocol=TCP --port=8080 --name=vamp --type="LoadBalancer"
```
{{% /copyable %}}

The Vamp image uses the following [configuration](https://github.com/magneticio/vamp-docker/blob/master/vamp-kubernetes/application.conf).

Now wait a bit until Vamp is running and check out the Kubernetes services:
{{% copyable %}}
```bash
kubectl get services
```
{{% /copyable %}}

The output should be similar to this:

```
NAME                 CLUSTER-IP     EXTERNAL-IP      PORT(S)             AGE
elasticsearch        10.3.242.188   <none>           9200/TCP            4m
etcd-client          10.3.247.112   <none>           2379/TCP            4m
etcd0                10.3.251.13    <none>           2379/TCP,2380/TCP   4m
etcd1                10.3.251.103   <none>           2379/TCP,2380/TCP   4m
etcd2                10.3.250.20    <none>           2379/TCP,2380/TCP   4m
kubernetes           10.3.240.1     <none>           443/TCP             5m
logstash             10.3.254.16    <none>           10001/UDP           4m
vamp                 10.3.242.93    146.148.118.45   8080/TCP            2m
vamp-gateway-agent   10.3.254.234   146.148.22.145   80/TCP              2m
```

Notice that the Vamp UI is exposed (in this example) on `http://146.148.118.45:8080`<br>

Let's continue and deploy the `sava` demo application:

{{% copyable %}}
```yaml
---
name: sava:1.0
gateways:
  9050: sava/port
clusters:
  sava:
    services:
      breed:
        name: sava:1.0.0
        deployable: magneticio/sava:1.0.0
        ports:
          port: 8080/http
      scale:
        cpu: 0.2       
        memory: 64MB
        instances: 1
```
{{% /copyable %}}

>**Note**: Be sure that your cluster has enough resources (CPU, memory), otherwise deployments will be in a pending state.

Once it's running we can check if all Vamp Gateway Agent services are up:
{{% copyable %}}
```bash
kubectl get services --show-labels -l vamp=gateway
```
{{% /copyable %}}

We can see that for each gateway a new service is created:

```
NAME                      CLUSTER-IP     EXTERNAL-IP     PORT(S)     AGE       LABELS
hex1f8c9a0157c9fe3335e9   10.3.243.199   104.155.24.47   9050/TCP    2m        lookup_name=a7ad6869e65e9c047f956cf7d1b4d01a89e
ef486,vamp=gateway
hex26bb0695e9a85ec34b03   10.3.245.85    23.251.143.62   40000/TCP   2m        lookup_name=6ace45cb2c155e85bd0c84123d1dab5a6cb
12c97,vamp=gateway
```

>**Note**: In this setup Vamp is deliberately configured to initiate the exposure of all gateway and VGA ports. This would not be the case if the default and recommended setting are used.

Now we can access our `sava` service on `http://104.155.24.47:9050`

The default Kubernetes service type can be set in configuration: `vamp.container-driver.kubernetes.service-type`, possible values are `LoadBalancer` or `NodePort`. 

We can also access gateways using virtual hosts. Vamp Gateway Agent service is on IP `146.148.22.145` in this example, so:
```bash
curl --resolve 9050.sava-1-0.vamp:80:146.148.22.145 -v http://9050.sava-1-0.vamp
```

You can now try out other [guides](/documentation/guides/).

>**Note**: Don't forget to [clean up](https://cloud.google.com/container-engine/docs/quickstart#clean-up) your Kubernetes cluster and firewall rules if you don't want to use them anymore.