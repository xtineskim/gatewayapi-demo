# K8 Gateway API Demo ⛩️

This repo is dedicated to showcasing the [Gateway API](https://gateway-api.sigs.k8s.io/).
This material was created for the co-located event ServiceMesh Con for KubeCon NA 2022 [talk](https://www.youtube.com/watch?v=ZcUn1tOixsU) that I delivered!

---
**Content**
- [K8 Gateway API Demo ⛩️](#k8-gateway-api-demo-️)
    - [Pre requisites](#pre-requisites)
    - [Set up](#set-up)
    - [1 - Prefix match and header edit](#1---prefix-match-and-header-edit)
    - [2 - Traffic Split](#2---traffic-split)
    - [3 - Add a new Gateway](#3---add-a-new-gateway)
    - [Resources](#resources)

---
### Pre requisites

* A Kubernetes cluster
* Istio, with the **minimal profile**
  * `istioctl install --set profile=minimal -y`

### Set up 

1. Install the Gateway API CRDs into your cluster.
```
kubectl apply -f 
https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.5.1/standard-install.yaml
```
2. Clone this repo.
```
git clone https://github.com/xtineskim/gatewayapi-demo
```
3. Deploy the well-known sample application, [`httpbin`](https://httpbin.org/#/). This will create a v1 and v2 deployment and service of the sample application in the default namespace.
```
kubectl apply -f gatewayapi-demo/0-setup/httpbin-v1-v2.yaml
```

4. Enable the sidecar injection to your `default` 
namespace (optional)
```
kubectl label ns default istio-injection=enabled --overwrite
```

5. Deploy your `Gateway` and `HTTPRoute`

```
kubectl apply -f  gatewayapi-demo/0-setup/gateway-api-demo.yaml
```
This Gateway is configured to use the [Istio Gateway Controller](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/)
4. Get the `EXTERNAL_IP` of your Gateway
```
kubectl get gateway -n istio-ingress
```
Try `curl`-ing the `EXTERNAL_IP`. You should be able to get a 200 response.
```
curl -s -I -HHost:httpbin.example.com "<YOUR_EXTERNAL_IP>/get"
```
### 1 - Prefix match and header edit

The `HTTPRoute` is where the majority of the logic is handled. You have 
the ability to add matching on a URI, and also apply fileters to modify 
the header.

Currently, the `HTTPRoute` only exposes the `/get` path. Try `curl`-ing `/headers`.
```
curl -s -I -HHost:httpbin.example.com "<YOUR_EXTERNAL_IP>/get"
```
Let's say that your app devs wanted to add another path to their app. All they would have to do is specify in the `HTTPRoute` to match the new path.
In 1-header/httproute.yaml, note the new match on the ` type: PathPrefix` for `/headers`.  Also note that this HTTPRoute is adding `hello: world` to the response headers.

Apply the HTTPRoute to access add a new headers.
```
kubectl apply -f gatewayapi-demo/1-header/httproute.yaml
```

Try `curl`-ing with `/headers`. You should now be able to access it. You should also see `hello:world` in it's response headers.
```
curl -s -I -HHost:httpbin.example.com "<YOUR_EXTERNAL_IP>/headers"
```

### 2 - Traffic Split

To split traffic between v1 and v2 of the the `httpbin` sample app, you can specify 
`weights` to state the percentage you want.
```
kubectl apply -f gatewayapi-demo/2-traffic-split/traffic-split-httproute.yaml
``` 
In this resource, you are specifying a 75%/25% split on v1 and v2.

If you `curl` your `EXTERNAL_IP/get` path, you should get a split between v1 and v2 according to the percentages!
```
curl -s -I -HHost:httpbin.example.com "<YOUR_EXTERNAL_IP>/get"
```
### 3 - Add a new Gateway

To try out another Gateway Controller (in this example, GKE L7 Load Balancer), you just need to instantiate your new 
Gateway with the appropriate `gatewayClassName`.

In the [`3-newGateway/`](./3-newGateway/) directory, the `gateway.yaml` contains the new GKE L7 LB. If you look at the [`httproute`](./3-Gateway/httproute.yaml), note the new entry in the `parentRefs` to allow traffic to flow from the new Gateway (in a different namespace) to the `httpbin` service.

First create a new namespace for the GKE L7
```
kubectl create ns gke-ingress
```
Create the new Gateway, and deploy the edited HTTPRoute
```
kubectl apply -f gatewayapi-demo/3-newGateway/gateway.yaml
kubectl apply -f gatewayapi-demo/3-newGateway/httproute.yaml
```

If you get the EXTERNAL_IP of your Gateways, you will now be able to interact with both Gateways and access the `httpbin` app!

### Resources

https://gateway-api.sigs.k8s.io/