# Deploying Custom Istio Ingress Gateways on IKS Managed Istio

### Istio 1.7 and 1.8 only. 

#### NOTE: Before upgrading from 1.7 to 1.8, ensure that your IstioOperator does not have a `revision` field.

Managed Istio on IKS comes with a `istio-ingressgateway` deployment in the `istio-system` namespace which you can use for routing traffic coming in to your mesh. 
```
$ kubectl get deploy -n istio-system
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
istio-egressgateway    2/2     2            2           138m
istio-ingressgateway   2/2     2            2           138m
istiod                 2/2     2            2           138m
```
It is exposed as a LoadBalancer service and is bound to a public IP address.
```
$ kubectl get svc -n istio-system istio-ingressgateway
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                                                                                                      AGE
istio-ingressgateway   LoadBalancer   172.21.191.13   169.63.158.243   15020:32189/TCP,80:32229/TCP,443:30727/TCP,15029:31567/TCP,15030:32151/TCP,15031:31829/TCP,15032:32446/TCP,31400:31871/TCP,15443:32171/TCP   139m
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
2. Modify Ingress Gateway with customizations.
3. Create an Ingress Gateway for private NLB traffic
4. Control version updates independently of Managed Istio updates. 

![](images/istioingress-custom3.png)

## Instructions
These steps will create a new Istio ingress gateway deployment in a `custom-gateways` namespace.

1. Enable Managed Istio 1.7
2. Create a namespace for the custom gateways.
```
kubectl create namespace custom-gateways
```
3. Create a file called `custom-ingress-iop.yaml` with contents:
```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: ibm-operators
  name: custom-ingressgateway-iop
spec:
  profile: empty
  hub: icr.io/ext/istio
  # tag: 1.7.1 # Force the Gateway to a specific version
  components:
    ingressGateways:
      - name: custom-ingressgateway
        label: 
          istio: custom-ingressgateway
        namespace: custom-gateways
        enabled: true
        k8s:
          serviceAnnotations:
            service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type: public # Change to private for a private IP
```
4. Apply the above `IstioOperator` CR to your cluster. The Managed Istio operator running in the `ibm-operators` namespace will read the resource and install the Gateway Deployment and Service into the `custom-gateways` namespace.
```
kubectl apply -f ./custom-ingress-iop.yaml
```
5. Check the deployments and services in `custom-gateways` namespace. You should see the new gateway deployed. 
```
kubectl get deploy,svc -n custom-gateways
```
```
NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/custom-ingressgateway   1/1     1            1           4m53s

NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                                                                                                      AGE
service/custom-ingressgateway   LoadBalancer   172.21.98.120   52.117.68.222   15020:32656/TCP,80:30576/TCP,443:32689/TCP,15029:31885/TCP,15030:30198/TCP,15031:32637/TCP,15032:30869/TCP,31400:30310/TCP,15443:31698/TCP   4m53s

```
6. Note the IP in the External-IP column. For VPC clusters, you will see a hostname. Register the IP address/hostname by creating a DNS subdomain: https://cloud.ibm.com/docs/containers?topic=containers-istio-mesh#istio_expose_bookinfo

### BookInfo sample

1. Create a new namespace and enable automatic sidecar injection
```
kubectl create namespace bookinfo
kubectl label namespace bookinfo  istio-injection=enabled
```
2. Deploy the bookinfo sample.
```
kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.7/samples/bookinfo/platform/kube/bookinfo.yaml
```
3. Deploy Gateway and Virtual Service. Create a file called `bookinfo-custom-gateway.yaml` with contents:
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: custom-ingressgateway ### use the new gateway you just created
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
Note that we are specifying `istio: custom-ingressgateway` in the Gateway.

4. Apply the Gateway and VirtualService resource
```
kubectl apply -f bookinfo-custom-gateway.yaml -n bookinfo
```
5. Get the EXTERNAL-IP of the `custom-ingressgateway` service in the `custom-gateways` namespace
```
kubectl get svc -n custom-gateways
```
6. Visit http://EXTERNAL-IP/productpage

## Exposing BookInfo by using an IBM-provided subdomain with TLS

1. Register the custom gateway IP address to create the host name and specify the `custom-gateway` namespace for the TLS secrets.
```
ibmcloud ks nlb-dns create classic --ip $EXTERNAL-IP --secret-namespace custom-gateways --cluster <cluster_name_or_id>
```

2. Wait until the secret is generated.
```
kubectl get secret -n custom-gateways

mycluster-af23f234rwr3asdfasdf-002   kubernetes.io/tls                     2      15m
```

3. Modify the Gateway in the `bookinfo-custom-gateway.yaml` you created in the previous section. 
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: custom-ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: mycluster-af23f234rwr3asdfasdf-002 # name of the secret
    hosts:
    - "*"
```

4. Apply the Gateway and VirtualService resource
```
kubectl apply -f bookinfo-custom-gateway.yaml -n bookinfo
```

5. Visit your subdomain. For example: https://mycluster-af23f234rwr3asdfasdf-002.us-south.containers.appdomain.cloud/productpage

## Creating an Ingress Gateway with private IP

VLAN provides subnets that are used to assign IP addresses to your worker nodes and public app services. By default, standard IKS clusters are connected to both [public and private VLANs](https://cloud.ibm.com/docs/containers?topic=containers-subnets#basics_vlans). The default Istio ingressgateway service will be assigned a public IP from the public VLAN.

Note: If you configure the worker nodes to only be connected to [private VLANs only](https://cloud.ibm.com/docs/containers?topic=containers-clusters), a private IP from the private VLAN is automatically created for default istio ingress gateway. 

To create a custom Istio ingress gateway and specify a private IP type, set `service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type: private` in `custom-ingress-io.yaml`.

## Control Istio Gateway updates and version

Managed Istio automatically applies patch (major.minor.patch) updates to the Gateway components. For example, when Managed Istio 1.7.2 is released, all the Istio control plane and gateway pods get updated from 1.7.1. to 1.7.2 automatically. The update is done using a rolling update strategy to avoid downtime. However, it might be desirable for you to control when the update occurs so that you can stop production traffic to the cluster during the upgrade to avoid any downtime. This also allows you to test for any potential regressions with the new version.

To prevent Managed Istio from automatically updating the Gateways, create a custom gateway using the instructions above, and specify the `tag` to a version at or below the control plane version as shown in `istioctl version`. 

When the control plane is updated by Managed Istio, you should update the custom gateways as soon as possible. Run `istioctl version` to get the control plane version and then update the `tag` in the IstioOperator to the same version and re-apply. 

Note:
- Do not set the tag to a version newer than the control plane. 
- Running older version of a Gateway can expose you to security vulnerabilities. 

## Other recommended customizations

Managed Istio Gateway is pre-configured with additional customization including minimum replicas, preStop lifecycle hooks for graceful shutdowns, improved affinity and edge node support. To achieve the same features for custom gateways, refer to the following IstioOperator.

```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: ibm-operators
  name: custom-ingressgateway-iop
spec:
  profile: empty
  hub: icr.io/ext/istio
#  tag: 1.7.1 # Force the Gateway to a specific version 
  components:
    ingressGateways:
      - name: custom-ingressgateway
        label:
          istio: custom-ingressgateway
        namespace: custom-gateways
        enabled: true
        k8s:
          serviceAnnotations:
            service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type: public
          hpaSpec:
            minReplicas: 2
          tolerations:   ## schedule onto edge nodes
          - key: dedicated
            value: edge
          overlays:
            - kind: Deployment
              name: custom-ingressgateway
              patches:
                - path: spec.template.spec.containers.[name:istio-proxy].lifecycle
                  value:
                    preStop:
                      exec:
                        command: ["sleep", "25"]
                - path: spec.template.spec.affinity
                  value:
                    podAntiAffinity:
                      preferredDuringSchedulingIgnoredDuringExecution:
                        - podAffinityTerm:
                            labelSelector:
                              matchExpressions:
                                - key: app
                                  operator: In
                                  values:
                                    - custom-ingressgateway
                            topologyKey: kubernetes.io/hostname
                          weight: 100
                    nodeAffinity:
                      preferredDuringSchedulingIgnoredDuringExecution:
                        - preference:
                            matchExpressions:
                              - key: dedicated
                                operator: In
                                values:
                                  - edge
                          weight: 100

```

For classic (non-VPC) clusters, to specify zone affinity in a multi-zone cluster, refer to the annotations documented [here](https://cloud.ibm.com/docs/containers?topic=containers-loadbalancer). These annotations can be added to the `serviceAnnotations` field.

```
        k8s:
          serviceAnnotations:
            service.kubernetes.io/ibm-load-balancer-cloud-provider-zone: "dal12"
```

## Disable the default Ingress Gateway

To remove the default `istio-ingressgateway` Ingress Gateway in the `istio-system` namespace created by Managed Istio, use the `managed-istio-custom` config map to set `istio-ingressgateway-public-1-enabled: false` as documented [here](https://cloud.ibm.com/docs/containers?topic=containers-istio#customize).


## Clean up

Remove the `IstioOperator` resource
```
kubectl delete -f ./custom-ingress-iop.yaml
```
