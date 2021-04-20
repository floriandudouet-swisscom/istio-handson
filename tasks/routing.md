# Routing Management

## Version 1: No stars for reviews

Let's start by having consistency in our service. We want to redirect all traffic to version "none" which has no stars. This represents the first iteration of our webservice. 

For that purpose you can make use of Istio's [Virtual Service](https://istio.io/latest/docs/concepts/traffic-management/#virtual-services) concept following the according [reference](https://istio.io/latest/docs/reference/config/networking/virtual-service/).

Use the below template and replace the FILL_HERE tags according to the first iteration of each service: v1 for productpage, none (for "no stars) for reviews, alpha for for ratings and v-nopull for details.

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: FILL_HERE
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: FILL_HERE
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: FILL_HERE
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: FILL_HERE
---
```

The filled in solution is at [virtualservice-v1-solution.yml](../solutions/virtualservice-v1-solution.yml).

## Version 2: specific path for the swisscom user

The previous step can be considered as a baseline, and corresponds to what one would have at the beginning of life a new application. 

Let's assume we just rolled out a new version with ratings represented as black stars under label `black`. One initial way to test it would be by only routing requests going for this new version to a specific subset of users belonging to a given group. We are going to simulate that for a given user `swisscom`.

Istio allows for routing based on the HTTP headers of the upcoming request. The productpage service adds a custom `end-user` header to all outbound requests to the reviews webservice when a user is logged in, thus we will use that header to redirect traffic from user `swisscom` to the reviews page of version `black`.

Use the [virtual service's HTTPMatchRequest](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMatchRequest) reference and update the reviews Virtual Service to redirect the `swisscom` user to the `black` version. You can use the following template as a base.

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - match:
    - FILL_HERE
    route:
    - destination:
        host: reviews
        subset: FILL_HERE
  - route:
    - destination:
        host: reviews
        subset: FILL_HERE
 ```

The solution can be found at [virtualservice-swisscom-solution.yml](../solutions/virtualservice-swisscom-solution.yml). 

Validate that when logged in as the `swisscom` user (no password), you see black stars for rating.

## Version 3: partial rollout

Another common usecase to validate a new version of a service is to redirect a percentage of the traffic to this new service. In this exercise, we are going to redirect 30% of the traffic received by the reviews service to the black version. This can be done using weight based routing.

Note that with Istio this is done at the routing level, so you can safely have multiple deployments at Kubernetes level with different versions: as long as the routing configuration is not updated, no traffic will be routed to new versions. 

Use the [HTTPRouteDestination](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRouteDestination) reference and the below template to redirect 30% of the reviews traffic to black.

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: FILL_HERE
      FILL_HERE
    - destination:
        host: reviews
        subset: FILL_HERE
      FILL_HERE
```

The solution can be found at [virtualservice-weight-solution.yml](../solutions/virtualservice-weight-solution.yml).

Validate that about 30% of the requests go to the black stars version by refreshing the main page multiple times.
