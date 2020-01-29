# Quick guide for creating traffic split with SMI and Linkerd

## TrafficSplit between backend services with Service Mesh Interface

Demo guide: 

Deploy cluster:

#set sp
az ad sp create-for-rbac -n smi-sp --skip-assignment
##output from above:
"appId": "7b211c00-e2f3-4465-a095-c1443b433761",
  "displayName": "smi-sp",
  "name": "http://smi-sp",
  "password": "xx",
  "tenant": "xx"

#create resource-group in Westeurope
az group create --name gustav-aks-linuxsummit-rg --location westeurope

#create aks cluster with windows nodes
az aks create --resource-group gustav-aks-smi-rg \
--name gustav-aks-linuxsummit \
--node-count 3 \
--kubernetes-version 1.14.7 \
--windows-admin-username azureuser \
--windows-admin-password Exclamati0n! \
--enable-vmss \
--generate-ssh-keys \
--network-plugin azure \
--service-principal xxx \
--client-secret xxx \
--node-zones {1,2,3} \
--debug



Demo script:
#Pre req
#1)Kubernetes cluster
#2)Internet access!External Access
#3)admin rights to install Linkerd and SMI spec
#4) CI/CD (if times allows it)*

##switch clusters:
##1) az aks get-credentials --name gustav-aks-mesh --resource-group gustav-aks-mesh-rg (new)
##2) az aks get-credentials --name gustav-aks-linuxsummit --resource-group gustav-aks-smi-rg (old)

##!!deploy linkerd into custom namespace (simple-service) with annotation and remember to delete/restart pods
kubectl annotate namespaces simple-service linkerd.io/inject=enabled

----------------------------------------------------------------------------------------------------

##Create namespace:
kubectl create namespace simple-service

##Create simple-service
kubectl apply -f simple-service-v1.yaml

##Run debug pod, which is ubuntu
kubectl apply -f debug-deployment.yaml

## View pod
k get pods

##Login to Ubuntu and update the service
 kubectl exec -it debug-xxxxxx bash
#root@debug-5f7d54889f-k5s67:/# 
apt update
#root@debug-5f7d54889f-k5s67:/
apt install curl
curl simple-service-v1.simple-service.svc.cluster.local.

### result success == I´m a service v1root

##Create the service (as root service for Traffic Split: https://github.com/deislabs/smi-spec/blob/master/traffic-split.md)
kubectl apply -f simple-service-root.yaml

##Login the Ubuntu pod and curl the new service
 kubectl exec -it debug-xxxx bash
curl simple-service.simple-service.svc.cluster.local.

##it works

##create the second service v2
kubectl apply -f simple-service-v2.yaml

##login Ubuntu and curl service v2
kubectl exec -it debug-xxxx bash

curl simple-service-v2.simple-service.svc.cluster.local.
###Result sucess == I'm new!! Service v2 

#curl root service multiable times to check if it's running - SPAM it!:
for i in {1..10};do curl simple-service.simple-service.svc.cluster.local.; echo; done

##The traffic split worked between v1 and v2, but it is NOT controlled!
##We also did not use version labels for selector
kubectl get svc/simple-service -n simple-service -o yaml 
#### THIS IS WHERE INGRESS CONTROLLERS or SERVICE MESH SIDECARS SPIT INCOMING TRAFFIC TO VARIOUS DESTINATIONS ####

##FIRST - INSTALL Linkerd in your K8s cluster
linkerd install | kubectl apply -f -
###... should be done in 3 min

#verify installation
linkerd check  

# Now we need to inject proxies pods into our service v1 and v2
kubectl get deploy -o yaml -n simple-service | linkerd inject -

# Now, MESH simple-service
kubectl get deploy -o yaml -n simple-service | linkerd inject - | kubectl apply -f -
#now you gained all of these features:  https://linkerd.io/2/features/ 
#mTLS, Blue/Green Canary deployment... etc.


##! If time allows it: kubectl describe pod/debug-5df9f65b74-2hvlg -n default (explain sidecars)
## we now have support for default mTLS - secure communication, telemetry and more: https://linkerd.io/2/features/ 
## Lets do some Canary release with linkerd deployed, pods are meshed and all we need is TrafficSplit

# look at for traffic spit details: https://github.com/deislabs/smi-spec/blob/master/traffic-split.md 

# add trafficsplit 
kubectl apply -f traffic-split.yaml

# view trafficsplit details
kubectl get trafficsplits.split.smi-spec.io -o yaml -n simple-service

#The trafficsplit declares that all traffic should go to simple-service-v1 https://github.com/deislabs/smi-spec/blob/master/traffic-split.md#tradeoffs
#Even if the simple-service root selector selects v2 pod, our SMI wont send traffic to it. Let's debug!

kubectl exec -it debug-5df9f65b74-2hvlg bash
for i in {1..10};do curl simple-service.simple-service.svc.cluster.local.; echo; done

#the outgoing traffic isnt being controlled by linkerd. Let us mesh it now!
kubectl get deploy -o yaml | linkerd inject - | kubectl apply -f -

#log into debug pod again (sorry for wait!)
kubectl exec -it debug-xxxx bash
apt update
apt install curl

for i in {1..10};do curl simple-service.simple-service.svc.cluster.local.; echo; done

#Let us edit trafficsplit weight
kubectl edit trafficsplits.split.smi-spec.io -o yaml -n simple-service

#let's verify configuration worked:
kubectl get trafficsplit -o yaml -n simple-service

# vNEXT
#1) Azure DevOps ~ GitOps demo!  
https://dev.azure.com/gustav-aks-devopsproject/AKS-SMI-ServiceMesh/_releaseDefinition?definitionId=1&_a=environments-editor-preview
#1.1) Demo: change weight (change cluster: 
az aks get-credentials --name gustav-aks-linuxsummit --resource-group gustav-aks-smi-rg
kubectl get trafficsplit -o yaml -n simple-service
linkerd edges po --all-namespaces

https://github.com/gkaleta/trafficsplit-linkerd-smi/blob/master/traffic-split.yaml 
#2) Canary release with Flagger - automating trafficsplit deployment based on weight / failure
https://docs.flagger.app/usage/linkerd-progressive-delivery
#3) Demo: linkerd dashboard &



Simple-servicev1.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: simple-service
    version: v1
  name: simple-service-v1
  namespace: simple-service
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: simple-service
        version: v1
    spec:
      containers:
      - image: jonathanbeber/simple-service:v1
        imagePullPolicy: IfNotPresent
        name: simple-service-v1
        ports:
          - containerPort: 8000
            name: http
            protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: simple-service-v1
  namespace: simple-service
spec:
  selector:
    app: simple-service
    version: v1
  ports:
  - protocol: TCP
    port: 80
    targetPort: http


simple-service-root.yaml
apiVersion: v1
kind: Service
metadata:
  name: simple-service
  namespace: simple-service
spec:
  selector:
    app: simple-service
  ports:
  - protocol: TCP
    port: 80
    targetPort: http
simple-service-v2.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: simple-service
    version: v2
  name: simple-service-v2
  namespace: simple-service
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: simple-service
        version: v2
    spec:
      containers:
      - image: jonathanbeber/simple-service:v2
        imagePullPolicy: IfNotPresent
        name: simple-service-v2
        ports:
          - containerPort: 8000
            name: http
            protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: simple-service-v2
  namespace: simple-service
spec:
  selector:
    app: simple-service
    version: v2
  ports:
  - protocol: TCP
    port: 80
    targetPort: http


debug-pod

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: debug
  name: debug
spec:
  replicas: 1
  selector:
    matchLabels:
      app: debug
  template:
    metadata:
      labels:
        app: debug
    spec:
      containers:
      - image: ubuntu
        name: ubuntu
        command:
        - sleep
        - "999999999999"


SMi-traffic.yaml

apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
    name: simple-service
    namespace: simple-service
spec:
    service: simple-service
    backends:
    - service: simple-service-v1
      weight: 1
    - service: simple-service-v2
      weight: 0



