#  Prerequisites

All these steps should be feasible on Windows, nonetheless a Linux/OSX system will make using some documentation a bit simpler.

## CLI tools to install
A note on the versions. These are the ones I tested on my Mac. Higher versions should work as long as they are backwards compatible.

Verify that you have the following tools handy on your system:

1. [minikube](https://minikube.sigs.k8s.io/docs/start/) version: v1.17.1
2. [kubectl](https://kubernetes.io/docs/tasks/tools/) version: v1.20.2

Run a test to verify your configuration: https://github.com/demosteveschmidt/techie-ws/blob/main/PREPARATION.md#run-a-quick-test

##  Istio preparation

1. Download the istio release compatible with your system: https://github.com/istio/istio/releases/tag/1.9.1
2. Move to the istio package directory: `cd istio-1.9.1`
3. Add `istioctl` to your path (`export PATH=$PWD/bin:$PATH` for Linux/OSX)
4. Verify that istioctl can be used: `istioctl version` should display the version.

## Istio deployment

```
istioctl install --set profile=demo -y

kubectl label namespace default istio-injection=enabled
```

Now all next steps are using the current Github and no longer the default Istio one.

1. Clone the repository: https://github.com/floriandudouet-swisscom/istio-handson.git
2. Deploy bookinfo

```
kubectl apply -f bookinfo/bookinfo.yaml
```
You want to see something similar to this list of pods:

```
taadufl1@UM00750 ~> kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
details-v-nopull-67748d86b8-m4dxw   2/2     Running   0          30s
fortio-deploy-687945c6dc-pf9bk      2/2     Running   0          43h
productpage-v1-66c6f7dd74-gxzkl     2/2     Running   0          29s
ratings-v-alpha-798d9665b5-jc2nh    2/2     Running   0          30s
reviews-black-7ff77cdf87-mtdqm      2/2     Running   0          30s
reviews-none-649dd75fbf-mp5xd       2/2     Running   0          30s
```

Create an istio gateway to configure external access into the app.

```
kubectl apply -f bookinfo/bookinfo-gateway.yaml
```

Verify the installation

```
istioctl analyze
```

Set up environment variables

```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```
Check port existence

```
echo "$INGRESS_PORT"
```

Setup host variable and verify existence

```
export INGRESS_HOST=$(minikube ip)
echo "$INGRESS_HOST"
```

Setup gateway variable

```
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
echo "http://$GATEWAY_URL/productpage"
```

Access the output of last command on your browser: you should see the bookinfo application. Refresh multiple times: you should see two different versions of the reviews, one with no stars rating and one with black stars.

We now setup observability tools. You will be able to experiment with them to see the outcome of some of the tasks. You may need to apply twice as some resources are created before their definition is ready.

```
kubectl apply -f observability/prometheus.yaml
kubectl apply -f observability/kiali.yaml
kubectl apply -f observability/grafana.yaml
kubectl apply -f observability/jaeger.yaml
```

Verify status with 

```
kubectl rollout status deployment/prometheus -n istio-system
kubectl rollout status deployment/kiali -n istio-system
kubectl rollout status deployment/grafana -n istio-system
kubectl rollout status deployment/jaeger -n istio-system
```

Access the dashboard with istioctl on another terminal (click on login)

```
istioctl dashboard kiali
```
Some data needs to be sent through the bookinfo app to get any graph to show

##   Destination Rules

We need to setup default [destination rules](https://istio.io/latest/docs/concepts/traffic-management/#destination-rules), which in particular are used to specify named service subsets, such as grouping all a given service’s instances by version, using the labels that have been setup on the services. 

These groups will be used by the Virtual Services as you will see in the following tasks.

Apply the default-destinations.yml rules file.

```
kubectl apply -f bookinfo/default-destinations.yml
```