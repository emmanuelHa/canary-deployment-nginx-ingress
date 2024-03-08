# Setup of a canary deploymnt with Kubernetes and Kubernetes Nginx Ingress Controller
Use case:
Imagine you need to test a new deployment of a new version of an application from 1.3 to 1.4, or 1.4 to 1.5.  
Imagine each application version has an endpoint you can use to assume the deployment is "ok".
A naive scenario could be to:
- loop for a while,
- curl a load balancer endpoint, 
- depending on which version was curled determine the error ratio for each version of the deployment (1.3 or 1.4 or 1.5)
- based on this ratio   
  if it is < threshold then deploy new version  
  if it is > threshold then remove the deployment of the version 

One solution could be to use an Nginx Ingress controller in front of Ingresses. Each ingress associated with one version   

Two providers of Nginx Ingress Controller exist:
- Nginx one: https://docs.nginx.com/nginx-ingress-controller/
- Kubernetes one: https://kubernetes.github.io/ingress-nginx/deploy/
Here the latter is used (not because it's the best one)

# Technologies:
- Kubernetes
- Docker / DockerHub

Prerequisites:
- be logged to DockerHub
- Kubernetes installed and started

## Create a new namespace:
```bash
kubectl create namespace ingress-nginx
```
## Install Nginx Ingress Controller

### Either from all in once modified manifest (version 1.9.5)
```bash
kubectl apply -f nginx-ingress.1.9.5.yaml
```

### Either from scratch
- you can install it with the help of Helm
  Follow the instructions: https://kubernetes.github.io/ingress-nginx/deploy/#quick-start
- Once applied you should modify the configMap ingress-nginx-controller in the namespace: ingress-nginx to add two annotations.
Doc is here: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/
```bash
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/docs/examples/customization/custom-configuration/configmap.yaml . 
```
Then add this snipet to the configMap:
```bash
data:
  allow-backend-server-header: "true"
  allow-snippet-annotations: "true"
  http-snippet: |
    map $proxy_alternative_upstream_name $my_service_name {
      "~^\w+-(.*)-\d+?" "$proxy_alternative_upstream_name";
      default "$proxy_upstream_name";
    }
```    

## Install 2 ingresses:
Example for version 1.3 and 1.4:
```bash
kubectl apply -f config-main-1.3.yaml
kubectl apply -f config-canary-1.4.yaml
```

## GET external IP of load balancer:
```bash
kubectl get svc ingress-nginx-controller  -n ingress-nginx

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   w.x.y.z         IP or localhost     80:31443/TCP,443:31189/TCP   5m
```

## Launch 10 curl every each seconds service version 1.3 and 1.4 and display error rates:
```bash
./test.sh 1 10 <external-ip>
```

# Clean up everything:
TODO: create a dedicated namespace for all resources 
```bash
kubectl delete -f nginx-ingress.1.9.5.yaml
kubectl delete -n my-custom-ns --all
```

# TODO
- Use another docker image  
- Bash is ugly to process curl responses  
- Canary testing can be done with gitlab-ci as well  


bash
docker run -p 3000:3000 blentai/hands-on-k8s-canary:1.3

See https://kubernetes.github.io/ingress-nginx/examples/canary/
to have an example how to configure a canary testing


Nice to read:

https://kubernetes.github.io/ingress-nginx/examples/customization/custom-headers/
https://kubernetes.github.io/ingress-nginx/examples/customization/custom-headers/configmap.yaml
https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/docs/examples/customization/custom-headers/custom-headers.yaml
https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/docs/examples/customization/custom-headers/configmap-client-response.yaml
