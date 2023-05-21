# notes

## Container Images

Container images are typically combined with a container configuration file,
which provides instructions on how to set up the container environment and
execute an application entry point. The container configuration often includes
information on how to set up networking, namespace isolation, resource
constraints (cgroups), and what syscall restrictions should be placed on a
running container instance. The container root filesystem and configuration file
are typically bundled using the Docker image format.

Containers fall into two main categories:

* System containers
* Application containers

System containers seek to mimic virtual machines and often run a full boot
process. They often include a set of system services typically found in a VM,
such as ssh, cron, and syslog. When Docker was new, these types of containers
were much more common. Over time, they have come to be seen as poor practice and
application containers have gained favor.

Application containers differ from system containers in that they commonly run a
single program. While running a single program per container might seem like an
unnecessary constraint, it provides the perfect level of granularity for
composing scalable applications and is a design philosophy that is leveraged
heavily by Pods. We will examine how Pods work in detail in Chapter 5.

## creating and running containers

```bash

# Note: if you are running on Windows you may need to fix line-endings using:
# --config core.autocrlf=input
git clone https://github.com/kubernetes-up-and-running/kuard
cd kuard
docker build -t kuard .
docker run --rm -p 8080:8080 kuard

docker login

docker tag kuard russomi/kuard-amd64:blue
docker push russomi/kuard-amd64:blue

docker run -d --name kuard \
  --publish 8080:8080 \
  russomi/kuard-amd64:blue

docker stop kuard
docker rm kuard

docker run -d --name kuard \
  --publish 8080:8080 \
  --memory 200m \
  --memory-swap 1G \
  russomi/kuard-amd64:blue

docker stop kuard
docker rm kuard

docker run -d --name kuard \
  --publish 8080:8080 \
  --memory 200m \
  --memory-swap 1G \
  --cpu-shares 1024 \
  russomi/kuard-amd64:blue

docker ps

docker images

docker rmi <tag-name>
docker rmi <image-id>

docker system prune
docker system prune -a

```

## Deploying a Kubernetes Cluster

```bash
https://github.com/kubernetes/minikube

minikube start
minikube stop
minikube delete

kubectl version

kubectl get componentstatuses
kubectl get nodes
kubectl describe nodes
kubectl describe nodes minikube

kubectl get deployments --namespace=kube-system coredns
kubectl get services --namespace=kube-system kube-dns

minikube addons list

```

## Common kubectl Commands

```bash

# namespaces

kubectl --namespace=mystuff
kubectl --all-namespaces

# context

kubectl config set-context my-context --namespace=mystuff
kubectl config use-context my-context

# get, describe, explain

kubectl get pods my-pod -o jsonpath --template={.status.podIP}
kubectl get pods,services
kubectl describe <resource-name> <obj-name>
kubectl explain pods
kubectl get pods --watch

# creating, updating, and destroying kubernetes objects

kubectl apply -f obj.yaml
kubectl edit <resource-name> <obj-name>
kubectl apply -f myobj.yaml view-last-applied
# edit-last-applied, set-last-applied, and view-last-applied
kubectl delete -f obj.yaml
kubectl delete <resource-name> <obj-name>

# labeling and annotating objects

kubectl label pods bar color=red
kubectl annotation pods bar color=red
# remove a label using <label-name>-
kubectl label pods bar color-

# debugging commands

kubectl logs <pod-name>
kubectl exec -it <pod-name> -- bash
kubectl attach -it <pod-name>
kubectl cp <pod-name>:</path/to/remote/file> </path/to/local/file>
kubectl port-forward <pod-name> 8080:80
kubectl get events
kubectl top nodes
kubectl top pods

# cluster management

kubectl cordon
kubectl drain
kubectl uncordon

# command completion

# see https://kubernetes.io/docs/tasks/tools/

# macOS
brew install bash-completion

# CentOS/Red Hat
yum install bash-completion

# Debian/Ubuntu
apt-get install bash-completion

source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ${HOME}/.bashrc

# help

kubectl help
kubectl help <command-name>

```

## pods

```bash
kubectl run kuard  \
  --image=russomi/kuard-amd64:blue
kubectl get pods
kubectl delete pods/kuard
kubectl get pods
kubectl apply -f 5-1-kuard-pod.yaml
kubectl describe pods kuard
kubectl logs kuard
kubectl logs kuard -f
kubectl logs kuard --previous
kubectl exec kuard -- date
kubectl exec -it kuard -- ash
kubectl apply -f 5-2-kuard-pod-health.yaml
kubectl port-forward kuard 8080:8080

kubectl delete pods/kuard
kubectl delete -f 5-1-kuard-pod.yaml
```

## labels and annotations

Annotations provide a place to store additional metadata for Kubernetes objects
where the sole purpose of the metadata is assisting tools and libraries.

> While labels are used to identify and group objects, annotations are used to
> provide extra information about where an object came from, how to use it, or
> policy around that object.
>
> There is overlap, and it is a matter of taste as to
> when to use an annotation or a label.
>
> When in doubt, add information to an
> object as an annotation and promote it to a label if you find yourself wanting
> to use it in a selector.

```bash

kubectl create deployment alpaca-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:blue

kubectl label deployments alpaca-prod ver=1 app=alpaca env=prod --overwrite=true

kubectl create deployment alpaca-test \
  --image=gcr.io/kuar-demo/kuard-amd64:green

kubectl label deployments alpaca-test ver=2 app=alpaca env=test --overwrite=true

kubectl create deployment bandicoot-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:green

kubectl label deployments bandicoot-prod ver=2 app=bandicoot env=prod --overwrite=true

kubectl create deployment bandicoot-staging \
  --image=gcr.io/kuar-demo/kuard-amd64:green

kubectl label deployments bandicoot-staging ver=2 app=bandicoot env=staging --overwrite=true

kubectl get deployments --show-labels
kubectl label deployments alpaca-test 'canary=true'
kubectl get deployments -L canary
kubectl get deployments --selector='ver=2'
kubectl get deployments --selector='app in (alpaca,bandicoot)'
kubectl get deployments --selector='canary'
kubectl get deployments --selector='!canary'
kubectl get deployments -l 'ver=2,!canary'
kubectl label deployments alpaca-test "canary-"

kubectl delete deployments --all
```

> Labels are used to identify and optionally group objects in a Kubernetes
> cluster. They are also used in selector queries to provide flexible runtime
> grouping of objects, such as Pods.

> Annotations provide object-scoped key/value metadata storage used by
> automation tooling and client libraries. They can also be used to hold
> configuration data for external tools such as third-party schedulers and
> monitoring tools.

> Labels and annotations are vital to understanding how key components in a
> Kubernetes cluster work together to ensure the desired cluster state. Using
> them properly unlocks the true power of Kubernetes’s flexibility and provides
> a starting point for building automation tools and deployment workflows.

## Service Discovery

> Service-discovery tools help solve the problem of finding which processes are
> listening at which addresses for which services.

```bash

kubectl create deployment alpaca-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:blue \
  --port=8080
kubectl scale deployment alpaca-prod --replicas 3
kubectl expose deployment alpaca-prod

kubectl create deployment bandicoot-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:green \
  --port=8080
kubectl scale deployment bandicoot-prod --replicas 2
kubectl expose deployment bandicoot-prod
kubectl get services -o wide

ALPACA_POD=$(kubectl get pods -l app=alpaca-prod \
    -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward $ALPACA_POD 48858:8080

kubectl edit deployment/alpaca-prod

        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 2
          initialDelaySeconds: 0
          failureThreshold: 3
          successThreshold: 1

ALPACA_POD=$(kubectl get pods -l app=alpaca-prod \
    -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward $ALPACA_POD 48858:8080

kubectl get endpoints alpaca-prod --watch

kubectl edit service alpaca-prod

ssh $(minikube ip) -L 8080:localhost:30129

kubectl get endpoints alpaca-prod --watch

kubectl delete deployment alpaca-prod
kubectl create deployment alpaca-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:blue \
  --port=8080
kubectl scale deployment alpaca-prod --replicas=3

kubectl get pods -o wide --selector=app=alpaca

kubectl get pods -o wide --show-labels

kubectl delete services,deployments -l app

```

## HTTP Load Balancing with Ingress

- Service object operates at Layer 4 (according to the OSI model)
- Kubernetes HTTP-based load-balancing system is called Ingress
- Ingress is split into a common resource specification and a
  controller implementation.
- There is no “standard” Ingress controller that is
  built into Kubernetes, so the user must install one of many optional
  implementations.

```bash
# You can install Contour with a simple one-line invocation:
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
kubectl get -n projectcontour service envoy -o wide
minikube tunnel

# If you are using minikube, you probably won’t have anything listed for EXTERNAL-IP.
# To fix this, you need to open a separate terminal window and run `minikube tunnel`.
# This configures networking routes such that you have unique IP addresses assigned
# to every service of type: LoadBalancer.

echo "10.98.49.91 alpaca.example.com bandicoot.example.com" >> /etc/hosts

# Using Ingress

kubectl create deployment be-default \
  --image=gcr.io/kuar-demo/kuard-amd64:blue \
  --replicas=3 \
  --port=8080

kubectl expose deployment be-default

kubectl create deployment alpaca \
  --image=gcr.io/kuar-demo/kuard-amd64:green \
  --replicas=3 \
  --port=8080

kubectl expose deployment alpaca

kubectl create deployment bandicoot \
  --image=gcr.io/kuar-demo/kuard-amd64:purple \
  --replicas=3 \
  --port=8080

kubectl expose deployment bandicoot

kubectl get services -o wide

# simple ingress

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  defaultBackend:
    service:
      name: alpaca
      port:
        number: 8080

kubectl apply -f simple-ingress.yaml

kubectl get ingress

kubectl describe ingress simple-ingress

# host-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  defaultBackend:
    service:
      name: be-default
      port:
        number: 8080
  rules:
  - host: alpaca.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: alpaca
            port:
              number: 8080

kubectl apply -f host-ingress.yaml

kubectl get ingress

kubectl describe ingress host-ingress

# path-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
  - host: bandicoot.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: bandicoot
            port:
              number: 8080
      - pathType: Prefix
        path: "/a/"
        backend:
          service:
            name: alpaca
            port:
              number: 8080

kubectl apply -f path-ingress.yaml

kubectl get ingress

kubectl describe ingress path-ingress

# Clean up

kubectl delete ingress host-ingress path-ingress simple-ingress
kubectl delete service alpaca bandicoot be-default
kubectl delete deployment alpaca bandicoot be-default

```

## Resources

- [Kubernetes Up and Running][3]
- [How To Remove Docker Images, Containers, and Volumes][1]
- [Markdown Guide - Basic Syntax][2]
- [kubectl Cheat Sheet][4]
- [High performance ingress controller for Kubernetes][5]

---

[1]:https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes
[2]:https://www.markdownguide.org/basic-syntax/
[3]:https://github.com/kubernetes-up-and-running
[4]:https://kubernetes.io/docs/reference/kubectl/cheatsheet/
[5]:https://projectcontour.io/
