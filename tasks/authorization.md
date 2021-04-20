
## Authorization

One common use case often mentioned with a service mesh is using it for firewalling purposes. 
Istio's [authorization features](https://istio.io/latest/docs/concepts/security/#authorization) provide mesh-, namespace-, and workload-wide access control for workloads within the mesh. It can act upon end-user to workload or service-to-service level and is based on the [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) resource. 

In this task we'll configure first HTTP authorization which, as we have deployed an HTTP based app, is the natural way of managing authorization, and then we'll replicate that authorization model using TCP authorization with ports directly.

For simplicity's sake in this session, we won't go into the mTLS authentication between services (which is enabled by default with the istio profile we selected at installation) and directly delve in what can be achieved once services can be authenticated with each other.

###  HTTP Authorization

By default there is no authorization needed between services, all traffic is open. This changes as soon as an Authorization policy is set (for instance namespace wide).

Let's start the task by blocking all communications by default. In istio, this works by creating a policy with an empty specification. Fill in the following template to create the "allow-nothing" Authorization policy in the default namespace.

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  FILL_HERE
spec:
  FILL_HERE
```

The solution can be found in [allow-nothing-solution.yml](../solutions/allow-nothing-solution.yml).

Refresh the bookinfo page, you should see a "RBAC: action denied" error message, indicating that the deny rule is enforced with no exception.

Let's now make the bookinfo page usable again: we start with the first service that needs to be accessed: productpage. Use the next template to create a productpage-viewer policy allowing GET access to the productpage service from all sources. Hint: use a rule `To`.

```
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "productpage-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  action: FILL_HERE
  rules: FILL_HERE
```

The solution can be found at [productpage-viewer-solution.yml](../solutions/productpage-viewer-solution.yml).

Check the application one more time, you should now be able to get to the main page but there should be errors fetching product details and reviews: the productpage service is in fact not allowed to reach the details and reviews services. Similarly, the reviews service is not allowed to reach the ratings service.

Use the following templates for these three services for GET, restricting access to only the services that need it. To identify a service, use [source principals](https://istio.io/latest/docs/reference/config/security/conditions/#supported-conditions). Hint: the principal for the productpage service account is `cluster.local/ns/default/sa/bookinfo-productpage` and for the reviews' service account `cluster.local/ns/default/sa/bookinfo-reviews`.

```
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "details-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - from:
    FILL_HERE
    to:
    - operation:
        methods: ["GET"]
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "reviews-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    FILL_HERE
    to:
    - operation:
        methods: ["GET"]
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "ratings-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
  - from:
    FILL_HERE
    to:
    - operation:
        methods: ["GET"]
```

The solution can be found at [authorization-http-solution.yml](../solutions/authorization-http-solution.yml).

With the additional policies in place, you should see the application fully functioning when retrieving it anonymously. Note that if you try to use the login function, you will be blocked: the previous policies only allow GET! As an additional task if you wish, update your policies to allow POST so that you are able to login.

Cleanup: to continue with the next task, remove all the ALLOW policies to avoid potential conflicts:

```
kubectl delete authorizationpolicy details-viewer
kubectl delete authorizationpolicy productpage-viewer
kubectl delete authorizationpolicy reviews-viewer
kubectl delete authorizationpolicy ratings-viewer
```

You should be back to the RBAC: access denied page you saw earlier.


### TCP Authorization

Let's now give a try to TCP layer authorization, we now want to allow access from the relevant sources to a single port. The bookinfo services all use port 9080.

Use the below template and the [port field reference](https://istio.io/latest/docs/reference/config/security/authorization-policy/#Operation) of the AuthorizationPolicy resource to allow access to productpage, reviews, ratings and details on port 9080

```
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "productpage-viewer-tcp"
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules: 
  - to: FILL_HERE
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "details-viewer-tcp"
  namespace: default
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - to: FILL_HERE
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "reviews-viewer-tcp"
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - to: FILL_HERE
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "ratings-viewer-tcp"
  namespace: default
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
  - to: FILL_HERE
```

The solution can be found at [authorization-tcp-solution.yml](../solutions/authorization-tcp-solution.yml).

If configured correctly, you should be able to access the application and have all services behaving correctly, with no errors fetching resources.

Note that Istio technically allows a mix and match of TCP and HTTP rules configuration, but the rule then becomes invalid: invalid ALLOW rules are ignored, whereas in DENY's case, the HTTP-only fields are ignored.

With this we conclude our short introduction to authorization policies with Istio.

Cleanup: 

```
kubectl delete authorizationpolicy details-viewer-tcp
kubectl delete authorizationpolicy productpage-viewer-tcp
kubectl delete authorizationpolicy reviews-viewer-tcp
kubectl delete authorizationpolicy ratings-viewer-tcp
```
