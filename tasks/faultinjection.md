#  Fault Injection

A common task in a microservice-based application is testing its resiliency in case of various issues. In this section we are going to see how Istio can be of help in this usecase, allowing for such tests without needing to change the application code. 

We are going to test two examples: artificially delay a request between two services, and artificially inject HTTP errors.

To start with, reset the virtual service for reviews to the state where the swisscom user goes to the black stars reviews version using the [solution](../solutions/virtualservice-swisscom-solution.yml) of the previous task.

## HTTP Delay fault

Using the following [reference](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPFaultInjection-Delay) as a guide, introduce a 7s fixed delay to all the request from user swisscom to the ratings service. 

According to the service's documentation there is a 10s timeout in the application code from reviews towards rating, thus a 7s delay should still be fine for the application's overall functionality.

Use the template below as guide:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: swisscom
    fault:
      FILL_HERE
    route:
    - destination:
        host: ratings
        subset: alpha
  - route:
    - destination:
        host: ratings
        subset: alpha
```

The solution can be found at [virtualservice-delay-solution.yml](../solutions/virtualservice-delay-solution.yml).

You should be observing that the application is in fact not functioning correctly: there is an error in fetching product reviews when logged in as swisscom user.

This is actually a bug that can be traced to another interaction: the productpage towards reviews connection. In fact, there is a 3s timeout in the application code of productpage that displays this error if reviews does not answer in time.

This sort of issues is something that can easily happen in a microservices based app where each service is deployed by different teams with different constraints. Using this delay injection feature, istio can be of help to find out these potential issues before a full rollout to production for all users.

If you lower the delay to 2.5s, the application should function correctly again. To allow for a 7s delay, you would need to modify the productpage application code and increase the timeout towards reviews. Apply the [virtualservice-2.5s-delay-solution.yml](../solutions/virtualservice-2.5s-delay-solution.yml) configuration to verify this hypothesis.

## HTTP Abort fault

This time we want to see how the application behaves if the ratings service answers all requests with a 500 error. To limit end user exposure, we want to restrict this test to the `swisscom` user once more. 

Use the [abort fault injection feature](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPFaultInjection-Abort) with the following template to apply this configuration.

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: swisscom
    fault:
      FILL_HERE
    route:
    - destination:
        host: ratings
        subset: alpha
  - route:
    - destination:
        host: ratings
        subset: alpha
```

If your configuration is correct, you should see in the application a "Ratings service is currently unavailable" message under the book reviews for the swisscom user (and only for him). This shows that the reviews code has been correctly written to handle this type of errors from the ratings service.

The solution can be found at [virtualservice-abort-solution.yml](../solutions/virtualservice-abort-solution.yml).

Cleanup: apply the [virtualservice-v1-solution.yml](../solutions/virtualservice-v1-solution.yml) again.
