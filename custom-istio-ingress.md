# Deploying your own Istio Ingress Gateway

### NOTE: Istio 1.4 ONLY! These instructions will not work for 1.5

Managed Istio on IKS comes with a `istio-ingressgateway` deployment in the `istio-system` namespace which you can use for routing traffic coming in to your mesh. 
```
~ 🏎  $ kubectl get deploy -n istio-system
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
grafana                  1/1     1            1           41d
istio-citadel            1/1     1            1           41d
istio-egressgateway      2/2     2            2           41d
istio-galley             1/1     1            1           41d
istio-ingressgateway     2/2     2            2           41d  <---
istio-pilot              1/1     1            1           41d
istio-policy             1/1     1            1           41d
istio-sidecar-injector   1/1     1            1           41d
istio-telemetry          1/1     1            1           41d
istio-tracing            1/1     1            1           41d
kiali                    1/1     1            1           41d
prometheus               1/1     1            1           41d
```
It is exposed as a LoadBalancer service and is bound to a public IP address.
```
~ 🏎  $ kubectl get svc -n istio-system istio-ingressgateway
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                                                                                                      AGE
istio-ingressgateway   LoadBalancer   172.21.243.213   52.111.66.222   15020:32168/TCP,80:30110/TCP,443:32149/TCP,15029:30484/TCP,15030:32043/TCP,15031:30774/TCP,15032:30929/TCP,15443:31217/TCP   41d
```
This Ingress Gateway is available to use for all workloads in the mesh via the `Gateway` resource. For example:
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # <--
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```
This means, all traffic coming into the mesh will flow thru this `istio-ingressgateway` deployment. 

![](images/istioingress-default.png)

### Additional Gateways

For the following reasons, some users choose to create additional ingress gateway deployments. 
1. Separate traffic flows between certain workloads or namespaces
2. Modify Ingress Gateway with customizations such as SDS.
3. Create an Ingress Gateway for private NLB traffic


## Instructions
These steps will create a new Istio ingress gateway deployment in a `bookinfo` namespace and then deploy the BookInfo sample application to the same namespace.

1. Download the Istio 1.4 release and add `istioctl` to your PATH https://istio.io/docs/setup/getting-started/#download
2. Create a new namespace and enable automatic sidecar injection
```
kubectl create namespace bookinfo
kubectl label namespace bookinfo  istio-injection=enabled
```
3. Run the following command from the istio folder to generate the new ingress Deployment, Service and ServiceAccount
```
helm template install/kubernetes/helm/istio/ \
  --namespace bookinfo \
  --set global.istioNamespace=istio-system \
  -x charts/gateways/templates/deployment.yaml \
  -x charts/gateways/templates/service.yaml  \
  -x charts/gateways/templates/serviceaccount.yaml \
  --set gateways.istio-ingressgateway.enabled=true \
  --set gateways.istio-egressgateway.enabled=false \
  --set gateways.istio-ingressgateway.labels.app=custom-istio-ingressgateway \
  --set gateways.istio-ingressgateway.labels.istio=custom-ingressgateway \
  > customingress.yaml
```
4. Apply the above resource:
```
kubectl apply -f customingress.yaml
```
5. Check the deployments and services in `bookinfo` namespace. 
```
kubectl get deploy,svc -n bookinfo
```
```
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
istio-ingressgateway   1/1     1            1           56m

NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                                                                                      AGE                                                                                                                    56m
service/istio-ingressgateway   LoadBalancer   172.21.169.93   52.117.68.220   15020:30828/TCP,80:30154/TCP,443:32100/TCP,15029:32603/TCP,15030:31219/TCP,15031:30466/TCP,15032:30081/TCP,15443:30068/TCP   56m
```
6. Deploy the bookinfo sample.
```
kubectl apply -n bookinfo -f ./samples/bookinfo/platform/kube/bookinfo.yaml
```
7. Deploy Gateway and Virtual Service. Create a file called `bookinfo-custom-gateway.yaml` with contents:
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: custom-ingressgateway # use the CUSTOM istio controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```
8. Apply the file
```
kubectl apply -f bookinfo-custom-gateway.yaml -n bookinfo
```
7. Get the EXTERNAL-IP of the `istio-ingressgateway` service in the `bookinfo` namespace
```
kubectl get svc -n bookinfo
```
8. Visit http://EXTERNAL-IP/productpage

![](images/istioingress-custom.png)


## Creating an Ingress Gateway per zone for HA



Replace the `helm template` command in the [instructions](#instructions) above with the following (replace `dal12` with your zone):
```
helm template install/kubernetes/helm/istio/ \
  --namespace bookinfo \
  --set global.istioNamespace=istio-system \
  -x charts/gateways/templates/deployment.yaml \
  -x charts/gateways/templates/service.yaml  \
  -x charts/gateways/templates/serviceaccount.yaml \
  --set gateways.istio-ingressgateway.enabled=true \
  --set gateways.istio-egressgateway.enabled=false \
  --set gateways.istio-ingressgateway.labels.app=custom-istio-ingressgateway \
  --set gateways.istio-ingressgateway.labels.istio=custom-ingressgateway \
  --set gateways.istio-ingressgateway.serviceAnnotations.'service\.kubernetes\.io/ibm-load-balancer-cloud-provider-zone'=dal12 \
  > customingress.yaml
```

## Creating an Ingress Gateway with private IP

Replace the `helm template` command in the [instructions](#instructions) above with the following:
```
helm template install/kubernetes/helm/istio/ \
  --namespace bookinfo \
  --set global.istioNamespace=istio-system \
  -x charts/gateways/templates/deployment.yaml \
  -x charts/gateways/templates/service.yaml  \
  -x charts/gateways/templates/serviceaccount.yaml \
  --set gateways.istio-ingressgateway.enabled=true \
  --set gateways.istio-egressgateway.enabled=false \
  --set gateways.istio-ingressgateway.labels.app=custom-istio-ingressgateway \
  --set gateways.istio-ingressgateway.labels.istio=custom-ingressgateway \
  --set gateways.istio-ingressgateway.serviceAnnotations.'service\.kubernetes\.io/ibm-load-balancer-cloud-provider-ip-type'=private \
  > customingress.yaml
```
