An ingress controller is a piece of software that provides reverse proxy, configurable traffic routing, and TLS termination for Kubernetes services. Kubernetes ingress resources are used to configure the ingress rules and routes for individual Kubernetes services. Using an ingress controller and ingress rules, a single IP address can be used to route traffic to multiple services in a Kubernetes cluster.

This article shows you how to deploy the NGINX ingress controller in an Azure Kubernetes Service (AKS) cluster. The cert-manager project is used to automatically generate and configure DigiCert certificates. Finally, two applications are run in the AKS cluster, each of which is accessible over a single IP address.
Create a basic ingress controller with internal network connectivity:
Below are the steps involve to create internal ingress controller :
Step 1: Create a namespace for your ingress resources

kubectl create namespace ingress-basic

Step 2:  Use Helm to deploy an NGINX ingress controller

helm install stable/nginx-ingress \
    --namespace ingress-basic \
    -f internal-ingress.yaml \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
    
Step 3: Create a file named internal-ingress.yaml using the following example manifest file. This example assigns 10.240.0.42 to the loadBalancerIP resource. Provide your own internal IP address for use with the ingress controller. Make sure that this IP address is not already in use within your virtual network.

controller:
  service:
    loadBalancerIP: 10.240.0.42
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
      
Step 4: When the Kubernetes load balancer service is created for the NGINX ingress controller, your internal IP address is assigned, as shown in the following example output:

$ kubectl get service -l app=nginx-ingress --namespace ingress-basic
NAME                                              TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
alternating-coral-nginx-ingress-controller        LoadBalancer   10.0.97.109   10.240.0.42   80:31507/TCP,443:30707/TCP   1m
alternating-coral-nginx-ingress-default-backend   ClusterIP      10.0.134.66   <none>        80/TCP                       1m     
     
Step 5: Create an ingress route 

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: ingress-basic
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: aks-helloworld
          servicePort: 80
        path: /(.*)
      - backend:
          serviceName: ingress-demo
          servicePort: 80
        path: /hello-world-two(/|$)(.*)
