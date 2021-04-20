# Istio

Rather than going by the documentation only, the workshop is meant to show you by example a few capabilities of Istio and how it can make several operations of an app's lifecycle easier, in the context of a microservices-based app. 

That said, it's important to have an idea of what Istio is and how it works before starting! This document does not aim to be a replacement for Istio's official documentation and is instead aiming to be a short introduction to its concepts. Links to the official Istio website are also provided where relevant.

## What is Istio?

Istio is a Service Mesh that layers onto existing distributed applications, with most of its features transparent to the application. 

A "Service Mesh" is a network of microservices that make up a distributed application. As the application grows in complexity, it becomes more and more complex to manage, with additional requirements such as A/B testing (when introducing a new version of one of the microservices), chaos testing (injecting errors to verify reliability), or end-to-end authentication (so that each service securely connects to the next one) coming up quickly. 

Istio aims to be a platform helping a developer solve these problems without the need to modify his application, instead using custom resources on Kubernetes which are then translated by Istio into service mesh configuration.


## How does it work?

Essentially, Istio is a network of proxies that are managed by a central configuration plane. A set of proxies (based on [Envoy](https://www.envoyproxy.io/)) intercepts traffic between each service before it is received by the application. It thus controls all communication between the services, as well as collecting metrics along the way.

Istiod is a single daemon that provides service discovery, configuration and certificate management. It converts the desired rules sent by the platform owner into Envoy-Specific configuration. 

![Istio's Architecture](imgs/istio-arch.svg)

[Learn more about Istio's architecture](https://istio.io/latest/docs/ops/deployment/architecture/).
## What can you do with Istio?
### Traffic Management

Using traffic routing rules to control the flow of traffic between services, Istio allows for service-level configuration of properties like circuit breakers or timeouts, and makes it straightforward to set up important tasks like A/B testing, canary rollouts, and staged rollouts with percentage-based traffic splits.

These capabilities rely on the use of the Envoy proxies intercepting traffic between each service of the mesh. The service mesh Pilot in short configures all relevant proxies within the mesh according to the configuration sent by the owner, using kubernetes custom resources instead of traditional proxy configuration.

In the tasks, we'll use mainly three traffic management API resources: 

#### Virtual Services

A virtual service lets you configure how requests are routed to a service within the mesh, building on the basic connectivity and discovery provided by Istio and your platform. Each virtual service consists of a set of routing rules that are evaluated in order, letting Istio match each given request to the virtual service to a specific real destination within the mesh. Your mesh can require multiple virtual services or none depending on your use case.

An example of Virtual Service:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myservice
spec:
  hosts:
  - myservice
  http:
  - match:
    - headers:
        origin:
          prefix: test-user
    route:
    - destination:
        host: myservice
        subset: v1
  - route:
    - destination:
        host: myservice
        subset: v2
```

This redirects traffic for myservice to v1 if the request comes with a header named origin with a value starting by test-user; and v2 otherwise.

Further details in the [intro to virtual services](https://istio.io/latest/docs/concepts/traffic-management/#virtual-services) and [reference](https://istio.io/latest/docs/reference/config/networking/virtual-service/).

#### Destination Rules

Destination rules are mainly used to specify named service subsets, such as grouping all a given service’s instances by version. You can then use these service subsets in the routing rules of virtual services to control the traffic to different instances of your services.

Destination rules also let you customize Envoy’s traffic policies when calling the entire destination service or a particular service subset, such as your preferred load balancing model, TLS security mode, or circuit breaker settings.

An example of Destination Rule:

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-service
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

This defines two subsets using service labels (v1, v2) that can each be used as subset in a virtual service. Traffic going to v1 will use a random load balancing, while specifically for v2, will use a round robin load balancing.

Further details in the [intro to destination rules](https://istio.io/latest/docs/concepts/traffic-management/#destination-rules) and [reference](https://istio.io/latest/docs/reference/config/networking/destination-rule/).


#### Gateways

Gateways are used to manage inbound and outbound traffic to and from the mesh. Gateway configurations are applied to standalone Envoy proxies that are running at the edge of the mesh, rather than sidecar Envoy proxies running alongside the service workloads.

An example of a Gateway:

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      credentialName: ext-host-cert
```

This configuration lets HTTPS traffic from ext-host.example.com into the mesh on port 443, but doesn’t specify any routing for the traffic. A VirtualService can then optionally be bound to a gateway to control the forwarding of traffic arriving at a particular host or gateway port.

Further details in the [intro to gateways](https://istio.io/latest/docs/concepts/traffic-management/#gateways) and [reference](https://istio.io/latest/docs/reference/config/networking/gateway/).

### Security

Istio aims to answer many security features needed by a microservice-based application:
* Traffic encryption, to defend against man-in-the-middle attacks
* Mutual authentication and fine-grained access policies, to enable a secure and flexible service access control

Security in Istio involves multiple components:

* A Certificate Authority (CA) for key and certificate management
* The configuration API server distributes to the proxies:
  * authentication policies
  * authorization policies
  * secure naming information
* Sidecar and perimeter proxies work as Policy Enforcement Points (PEPs) to secure communication between clients and servers.

#### Authentication

Istio offers [mutual TLS](https://en.wikipedia.org/wiki/Mutual_authentication) as a full stack solution for traffic authentication between peers within the mesh. This is used for service-to-service authentication.

It also integrates with external OpenID providers to enable the verification of credentials for end-user authentication.

Further details on Authentication in [Istio's documentation](https://istio.io/latest/docs/concepts/security/#authentication).

#### Authorization

Istio’s authorization features provide mesh-, namespace-, and workload-wide access control for workloads within the mesh.

* Workload-to-workload and end-user-to-workload authorization.
* A single AuthorizationPolicy CRD, for all authorization purposes.
* Operators can define custom conditions on Istio attributes, and use DENY and ALLOW actions.

The authorization policy enforces access control to the inbound traffic in the server side Envoy proxy. Each Envoy proxy runs an authorization engine that authorizes requests at runtime. When a request comes to the proxy, the authorization engine evaluates the request context against the current authorization policies, and returns the authorization result, either ALLOW or DENY. Operators specify Istio authorization policies using .yaml files to deploy AuthorizationPolicy resources.

To configure an authorization policy, the operator creates an AuthorizationPolicy custom resource, with a selector, an action, and a list of rules:

* The selector field specifies the target of the policy
* The action field specifies whether to allow or deny the request
* The rules specify when to trigger the action
* The from field in the rules specifies the sources of the request
* The to field in the rules specifies the operations of the request
* The when field specifies the conditions needed to apply the rule

Here's an example of an AuthorizationPolicy:

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: httpbin
 namespace: foo
spec:
 selector:
   matchLabels:
     app: httpbin
     version: v1
 action: ALLOW
 rules:
 - from:
   - source:
       principals: ["cluster.local/ns/default/sa/sleep"]
   - source:
       namespaces: ["dev"]
   to:
   - operation:
       methods: ["GET"]
   when:
   - key: request.auth.claims[iss]
     values: ["https://accounts.google.com"]
```

It creates a `httpbin` AuthorizationPolicy in the `foo` namespace, that allows two sources, the `cluster.local/ns/default/sa/sleep` service account and the `dev` namespace to access the workloads with the `app: httpbin` and `version: v1` labels within the `foo` namespace when requests sent have a valid JWT token.

Further details on Authorization in [Istio's documentation](https://istio.io/latest/docs/concepts/security/#authorization) and [AuthorizationPolicy reference](https://istio.io/latest/docs/reference/config/security/authorization-policy/).