## An Introduction to Kubernetes and Rancher

Kubernetes has taken over the market. It doesn't matter if you believe in it or not; if you don't learn these skills, you will find yourself unable to compete for jobs against people who know how to use it.

Let's also have a look at the momentum that Kubernetes has been picking up -

* K8s' scalability and robust design patterns have made it the platform of choice for the industry
* Large software companies like Github have publicized their use of K8s
* Many major IT vendors have recently added K8s offerings

But despite all this momentum, Kubernetes is not without it's challenges. These are some real concerns that Kubernetes users have on a daily basis -

* With so, so many ways to deploy, how do I deploy consistently across different infrastructures?
* How do I implement and manage access control across multiple clusters and namespaces?
* How do I integrate with a central authentication system?
* How do I partition clusters to more efficiently use my resources?
* How do I manage multi-tenancy, multiple dedicated and shared clusters?
* How do I make my clusters highly available?
* How do I ensure that security policies are enforced across clusters/namespaces?
* Do I have sufficient visibility to detect and troubleshoot issues?

As an engineer, you want to make sure that whatever you choose today not only satisfies your requirements for today but has the flexibility to adapt as needs change in the future.

* * *

### The Case for Rancher

**Rancher** is a complete container management platform that allows you to launch instances and install Kubernetes onto them. It allows you to deploy and manage hosted Kubernetes solutions in any provider (e.g. AWS EKS, GCP GKE, Azure AKS, Ubuntu UKS, VMWare PKS, etc.). If you have a Kubernetes cluster in the wild, Rancher will allow you to import it - where your Kubernetes comes from does not matter, it's a commodity, Rancher can manage it. How does Rancher manage it, you ask? Rancher let's you corral (gather together and confine) all of your clusters under a single system under one highly available point of control and you don't have to sacrifice any of the individual cluster independence. This is important as what it means is that you can bundle namespaces into projects, you can map strong authorization roles that grant the correct level of access to those projects and you can restrict those projects and the resources inside of them from communicating or even be able to see anything else that's running in the cluster - single cluster multi-tenancy (tenants are logically isolated, but physically integrated). You can plug it into your existing active directory or SAML infrastructure. You can also apply that central authentication and authorization to all of your Kubernetes clusters running anywhere which means that you can readily see and control capacity and thus you can control your costs. You can scale infrastructure up or down based on metrics that you control. If you are looking for monitoring and logging, you can plug it into the existing metric server or the logging server or you can plug it into any external system that you want. If you want traditional access via kube control that's all still there and everything adheres to the role-based access control to be defined. If you're looking for CI/CD, you can plug Kubernetes or Rancher into any CI/CD system that you want. So now as a developer, you push your code and can spin up entire environments for testing application onto them, bang on them for a little while for some UAT or functional testing or whatever other testing you want to do and then tear all those clusters down entirely controlled from automation within Rancher. Rancher gives you total freedom and it's completely free and open source.

* * *

### Kubernetes 101

Basic building blocks of Kubernetes:

* Pods
* Deployments
* Services
* Config Maps
* Ingresses

#### Pods

Pods collect containers into a shared unit with combined network space and storage.

* Smallest unit that can be deployed in Kubernetes
* Consists of one or more containers that are always scheduled together
* Each pod is given a unique IP address
* Containers can speak to each other via localhost

You can do other cool stuff like attach other containers to the pod that do other things and it makes it really easy to adhere to best practices.

**Basic Pod Spec**

pod.yaml

```
apiVersion: v1
kind: Pod
metadata:
    name: myapp-pod
    labels:
        app: myapp
spec:
    containers:
    - name: myapp-container
      image: busybox
      command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 10']
```

You use a command called **kubectl** to apply the manifest to the cluster. Given above is a basic manifest for a single pod. It has all four required keys for a manifest which are apiVersion, kind, metadata and spec. The defined pod above is just gonna echo some text, sleep for 10 seconds and then exit.

```
kubectl apply -f pod.yaml
```

You can then watch your pod by doing

```
watch -n 1 kubectl get po
```

Kubernetes understands whn you tell it to run something, it's supposed to keep it running. So the pod and the container inside of it exited and Kubernetes was like oh that's not supposed to happen and it restarted it and ten seconds later it exited again and then went like oh that's not supposed to happen and it restarted it again and this continues until eventually Kubernetes realises that this thing is broken at which point it stops trying to restart it as frequently. Kubernetes is smart enough to figure out well something here is not right so I'm going to back off on this because there's some kind of a what they call it a crash loop.

You're not usually going to launch pods. Let's move on and see what a Replica Set and Deployments are and then we'll tie it back together.

#### Replica Set

* Defines the desired scale and state of a group of pods (e.g. If you tell it that you want three replicas, which means three copies of a pod, and if there's none - Kubernetes is gonna do it's best to make three of them appear)

![](/assets/images/replicaset.png)