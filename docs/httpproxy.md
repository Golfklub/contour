# HTTPProxy

The [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) object was added to Kubernetes in version 1.1 to describe properties of a cluster-wide reverse HTTP proxy.
Since that time, the Ingress object has not progressed beyond the beta stage, and its stagnation inspired an [explosion of annotations](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md) to express missing properties of HTTP routing.

The goal of the `HTTPProxy` (previously `IngressRoute`) Custom Resource Definition (CRD) is to expand upon the functionality of the Ingress API to allow for a richer user experience as well addressing the limitations of the latter's use in multi tenent environments.

## Key HTTPProxy Benefits

- Safely supports multi-team Kubernetes clusters, with the ability to limit which Namespaces may configure virtual hosts and TLS credentials.
- Enables including of routing configuration for a path or domain from another HTTPProxy, possibly in another Namespace.
- Accepts multiple services within a single route and load balances traffic across them.
- Natively allows defining service weighting and load balancing strategy without annotations.
- Validation of HTTPProxy objects at creation time and status reporting for post-creation validity.

## Ingress to HTTPProxy

A minimal Ingress object might look like:

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: basic
spec:
  rules:
  - host: foo-basic.bar.com
    http:
      paths:
      - backend:
          serviceName: s1
          servicePort: 80
```

This Ingress object, named `basic`, will route incoming HTTP traffic with a `Host:` header for `foo-basic.bar.com` to a Service named `s1` on port `80`.
Implementing similar behavior using an HTTPProxy looks like this:

```yaml
# httpproxy.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: basic
spec:
  virtualhost:
    fqdn: foo-basic.bar.com
  routes:
    - conditions:
      - prefix: /
      services:
        - name: s1
          port: 80
```

**Lines 1-4**: As with all other Kubernetes objects, an HTTPProxy needs apiVersion, kind, and metadata fields. Note that the HTTPProxy API is currently considered beta.

**Line 6-7**: The presence of the `virtualhost` field indicates that this is a root HTTPProxy that is the top level entry point for this domain.
The `fqdn` field specifies the fully qualified domain name that will be used to match against `Host:` HTTP headers.

**Lines 8-9**: Each HTTPProxy must have one or more `routes`, each of which must have a condition to match against (e.g. `prefix: /`) and then one or more `services` which will handle the HTTP traffic.

**Lines 10-12**: The `services` field is an array of named Service & Port combinations that will be used for this HTTPProxy path.
HTTP traffic will be sent directly to the Endpoints corresponding to the Service.

## Interacting with HTTPProxies

As with all Kubernetes objects, you can use `kubectl` to create, list, describe, edit, and delete HTTPProxy CRDs.

Creating an HTTPProxy:

```bash
$ kubectl create -f basic.httpproxy.yaml
httpproxy "basic" created
```

Listing HTTPProxies:

```bash
$ kubectl get httpproxy
NAME      AGE
basic     24s
```

Describing HTTPProxy:

```bash
$ kubectl describe httpproxy basic
Name:         basic
Namespace:    default
Labels:       <none>
API Version:  projectcontour.io/v1alpha1
Kind:         HTTPProxy
Metadata:
  Cluster Name:
  Creation Timestamp:  2019-07-05T19:26:54Z
  Resource Version:    19373717
  Self Link:           /apis/projectcontour.io/v1alpha1/namespaces/default/httpproxy/basic
  UID:                 6036a9d7-8089-11e8-ab00-f80f4182762e
Spec:
  Routes:
    Conditions:
      Prefix: /
    Services:
      Name:  s1
      Port:  80
  Virtualhost:
    Fqdn:  foo-basic.bar.com
Events:    <none>
```

Deleting HTTPProxies:

```bash
$ kubectl delete httpproxy basic
httpproxy "basic" deleted
```

## HTTPProxy API Specification

There are a number of [working examples](/examples/example-workload/httpproxy) of HTTPProxy objects in the `examples/example-workload` directory.

We will use these examples as a mechanism to describe HTTPProxy API functionality.

### Virtual Host Configuration

#### Fully Qualified Domain Name

Similar to Ingress, HTTPProxy support name-based virtual hosting.
Name-based virtual hosts use multiple host names with the same IP address.

```
foo.bar.com --|                 |-> foo.bar.com s1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com s2:80
```

Unlike Ingress, HTTPProxy only support a single root domain per HTTPProxy object.
As an example, this Ingress object:

```yaml
# ingress-name.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: name-example
spec:
  rules:
  - host: foo1.bar.com
    http:
      paths:
      - backend:
          serviceName: s1
          servicePort: 80
  - host: bar1.bar.com
    http:
      paths:
      - backend:
          serviceName: s2
          servicePort: 80
```

must be represented by two different HTTPProxy objects:

```yaml
# httpproxy-name.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: name-example-foo
  namespace: default
spec:
  virtualhost:
    fqdn: foo1.bar.com
  routes:
    - services:
      - name: s1
        port: 80
---
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: name-example-bar
  namespace: default
spec:
  virtualhost:
    fqdn: bar1.bar.com
  routes:
    - services:
        - name: s2
          port: 80
```

#### TLS

HTTPProxy follows a similar pattern to Ingress for configuring TLS credentials.

You can secure a HTTPProxy by specifying a Secret that contains TLS private key and certificate information.
Contour (via Envoy) uses the SNI TLS extension to handle this behavior.
If multiple HTTPProxy's utilize the same Secret, the certificate must include the necessary Subject Authority Name (SAN) for each fqdn.

Contour also follows a "secure first" approach.
When TLS is enabled for a virtual host any request to the insecure port is redirected to the secure interface with a 301 redirect.
Specific routes can be configured to override this behavior and handle insecure requests by enabling the `spec.routes.permitInsecure` parameter on a Route.

The TLS secret must contain keys named tls.crt and tls.key that contain the certificate and private key to use for TLS, e.g.:

```yaml
# ingress-tls.secret.yaml
apiVersion: v1
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
kind: Secret
metadata:
  name: testsecret
  namespace: default
type: kubernetes.io/tls
```

The HTTPProxy can be configured to use this secret using `tls.secretName` property:

```yaml
# httpproxy-tls.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: tls-example
  namespace: default
spec:
  virtualhost:
    fqdn: foo2.bar.com
    tls:
      secretName: testsecret
  routes:
    - services:
        - name: s1
          port: 80
```

If the `tls.secretName` property contains a slash, eg. `somenamespace/somesecret` then, subject to TLS Certificate Delegation, the TLS certificate will be read from `somesecret` in `somenamespace`.
See TLS Certificate Delegation below for more information.

The TLS **Minimum Protocol Version** a vhost should negotiate can be specified by setting the `spec.virtualhost.tls.minimumProtocolVersion`:

- 1.3
- 1.2
- 1.1 (Default)

#### Upstream TLS

TODO(youngnick): Update this once #1506 is delivered

A HTTPProxy can proxy to an upstream TLS connection by first annotating the upstream Kubernetes service with: `contour.heptio.com/upstream-protocol.tls: "443,https"`.
This annotation tells Contour which port should be used for the TLS connection.
In this example, the upstream service is named `https` and uses port `443`.
Additionally, it is possible for Envoy to verify the backend service's certificate.
The service of an HTTPProxy can optionally specify a `validation` struct which has a manditory `caSecret` key as well as an mandatory `subjectName`.

Note: If `spec.routes.services[].validation` is present, `spec.routes.services[].{name,port}` must point to a Service with a matching `projectcontour.io/upstream-protocol.tls` Service annotation.

##### Sample YAML

```yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: secure-backend
spec:
  virtualhost:
    fqdn: www.example.com  
  routes:
    - services:
        - name: service
          port: 8443
          validation:
            caSecret: my-certificate-authority
            subjectName: backend.example.com
```

##### Error conditions

If the `validation` spec is defined on a service, but the secret which it references does not exist, Contour will reject the update and set the status of the HTTPProxy object accordingly.
This helps prevent the case of proxying to an upstream where validation is requested, but not yet available.

```yaml
Status:
  Current Status:  invalid
  Description:     route "/": service "tls-nginx": upstreamValidation requested but secret not found or misconfigured
```

#### TLS Certificate Delegation

In order to support wildcard certificates, TLS certificates for a `*.somedomain.com`, which are stored in a namespace controlled by the cluster administrator, Contour supports a facility known as TLS Certificate Delegation.
This facility allows the owner of a TLS certificate to delegate, for the purposes of referencing the TLS certificate, permission to Contour to read the Secret object from another namespace.

```yaml
apiVersion: projectcontour.io/v1alpha1
kind: TLSCertificateDelegation
metadata:
  name: example-com-wildcard
  namespace: www-admin
spec:
  delegations:
    - secretName: example-com-wildcard
      targetNamespaces:
      - example-com
---
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: www
  namespace: example-com
spec:
  virtualhost:
    fqdn: foo2.bar.com
    tls:
      secretName: www-admin/example-com-wildcard
  routes:
    - services:
        - name: s1
          port: 80
```

In this example, the permission for Contour to reference the Secret `example-com-wildcard` in the `admin` namespace has been delegated to HTTPProxy objects in the `example-com` namespace.

### Conditions

Each Route entry in a HTTPProxy may contain one or more conditions.
These conditions are combined with an AND operator on the route passed to Envoy.

Conditions can be either a `prefix` or a `header` condition.
For `prefix`, this adds a path prefix.

#### Header conditions

For `header` conditions there is one required field, `name`, and five operator fields: `present`, `contains`, `notcontains`, `exact`, and `notexact`.

- `present` is a boolean and checks that the header is present. The value will not be checked.

- `contains` is a string, and checks that the header contains the string. `notcontains` similarly checks that the header does *not* contain the string.

- `exact` is a string, and checks that the header exactly matches the whole string. `notexact` checks that the header does *not* exactly match the whole string.

#### Multiple Routes

HTTPProxy must have at least one route or include defined.
Paths defined are matched using prefix conditions.
In this example, any requests to `multi-path.bar.com/blog` or `multi-path.bar.com/blog/*` will be routed to the Service `s2`.
All other requests to the host `multi-path.bar.com` will be routed to the Service `s1`.

```yaml
# httpproxy-multiple-paths.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: multiple-paths
  namespace: default
spec:
  virtualhost:
    fqdn: multi-path.bar.com
  routes:
    - conditions:
      - prefix: / # matches everything else
      services:
        - name: s1
          port: 80
    - conditions:
      - prefix: /blog # matches `multi-path.bar.com/blog` or `multi-path.bar.com/blog/*`
      services:
        - name: s2
          port: 80
```

In the following example, we match on headers and send to different services, with a default route if those do not match.

```yaml
# httpproxy-multiple-headers.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: multiple-paths
  namespace: default
spec:
  virtualhost:
    fqdn: multi-path.bar.com
  routes:
    - conditions:
      - header:
          name: x-os
          contains: ios
      services:
        - name: s1
          port: 80
    - conditions:
      - header:
          name: x-os
          contains: android
      services:
        - name: s2
          port: 80
    - services:
        - name: s3
          port: 80
```

#### Multiple Upstreams

One of the key HTTPProxy features is the ability to support multiple services for a given path:

```yaml
# httpproxy-multiple-upstreams.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: multiple-upstreams
  namespace: default
spec:
  virtualhost:
    fqdn: multi.bar.com
  routes:
    - services:
        - name: s1
          port: 80
        - name: s2
          port: 80
```

In this example, requests for `multi.bar.com/` will be load balanced across two Kubernetes Services, `s1`, and `s2`.
This is helpful when you need to split traffic for a given URL across two different versions of an application.

#### Upstream Weighting

Building on multiple upstreams is the ability to define relative weights for upstream Services.
This is commonly used for canary testing of new versions of an application when you want to send a small fraction of traffic to a specific Service.

```yaml
# httpproxy-weight-shfiting.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: weight-shifting
  namespace: default
spec:
  virtualhost:
    fqdn: weights.bar.com
  routes:
    - services:
        - name: s1
          port: 80
          weight: 10
        - name: s2
          port: 80
          weight: 90
```

In this example, we are sending 10% of the traffic to Service `s1`, while Service `s2` receives the remaining 90% of traffic.

HTTPProxy weighting follows some specific rules:

- If no weights are specified for a given route, it's assumed even distribution across the Services.
- Weights are relative and do not need to add up to 100. If all weights for a route are specified, then the "total" weight is the sum of those specified. As an example, if weights are 20, 30, 20 for three upstreams, the total weight would be 70. In this example, a weight of 30 would receive approximately 42.9% of traffic (30/70 = .4285).
- If some weights are specified but others are not, then it's assumed that upstreams without weights have an implicit weight of zero, and thus will not receive traffic.

#### Request Timeout

Each Route can be configured to have a timeout policy and a retry policy as shown:

```yaml
# httpproxy-request-timeout.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: request-timeout
  namespace: default
spec:
  virtualhost:
    fqdn: timeout.bar.com
  routes:
  - timeoutPolicy:
      request: 1s
    retryPolicy:
      count: 3
      perTryTimeout: 150ms
    services:
    - name: s1
      port: 80
```

In this example, requests to `timeout.bar.com/` will have a request timeout policy of 1s.
This refers to the time that spans between the point at which complete client request has been processed by the proxy, and when the response from the server has been completely processed.

- `timeoutPolicy.response` This field can be any positive time period or "infinity".
The time period of **0s** will also be treated as infinity.
This timeout is for how long it takes for the *backend service* to respond.
By default, Envoy has a 15 second timeout for this timeout.
More information can be found in [Envoy's documentation](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto.html#envoy-api-field-route-routeaction-timeout).
- `timeoutPolicy.idle` This field can be any positive time period or "infinity".
The time period of **0s** will also be treated as infinity.
By default, there is no per-route idle timeout.
Note that the default connection manager timeout of 5 minutes will apply if this is not set.
More information can be found in [Envoy's documentation](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto.html#envoy-api-field-route-routeaction-idle-timeout)
- `retryPolicy`: A retry will be attempted if the server returns an error code in the 5xx range, or if the server takes more than `retryPolicy.perTryTimeout` to process a request.
  - `retryPolicy.count` specifies the maximum number of retries allowed. This parameter is optional and defaults to 1.
  - `retryPolicy.perTryTimeout` specifies the timeout per retry. If this field is greater than the request timeout, it is ignored. This parameter is optional.
  If left unspecified, `timeoutPolicy.request` will be used.

#### Load Balancing Strategy

Each upstream service can have a load balancing strategy applied to determine which of its Endpoints is selected for the request.
The following list are the options available to choose from:

- `RoundRobin`: Each healthy upstream Endpoint is selected in round robin order (Default strategy if none selected).
- `WeightedLeastRequest`: The least request strategy uses an O(1) algorithm which selects two random healthy Endpoints and picks the Endpoint which has fewer active requests. Note: This algorithm is simple and sufficient for load testing. It should not be used where true weighted least request behavior is desired.
- `Random`: The random strategy selects a random healthy Endpoints.

More information on the load balancing strategy can be found in [Envoy's documentation](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing.html).

The following example defines the strategy for Service `s2-strategy` as `WeightedLeastRequest`.
Service `s1-strategy` does not have an explicit strategy defined so it will use the strategy of `RoundRobin`.

```yaml
# httpproxy-lb-strategy.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: lb-strategy
  namespace: default
spec:
  virtualhost:
    fqdn: strategy.bar.com
  routes:
    - conditions:
      - prefix: /
      services:
        - name: s1-strategy
          port: 80
        - name: s2-strategy
          port: 80
          strategy: WeightedLeastRequest
```

#### Session Affinity

Session affinity, also known as _sticky sessions_, is a load balancing strategy whereby a sequence of requests from a single client are consitently routed to the same application backend.
Contour supports session affinity with the `strategy: Cookie` key on a per service basis.

```yaml
# httpproxy-sticky-sessions.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: httpbin
  namespace: default
spec:
  virtualhost:
    fqdn: httpbin.davecheney.com
  routes:
  - services:
    - name: httpbin
      port: 8080
      strategy: Cookie
```

##### Limitations

Session affinity is based on the premise that the backend servers are robust, do not change ordering, or grow and shrink according to load.
None of these properties are guaranteed by a Kubernetes cluster and will be visible to applications that rely heavily on session affinity.

Any perturbation in the set of pods backing a service risks redistributing backends around the hash ring.

Active health checking can be configured on a per-upstream Service basis.
Contour supports HTTP health checking and can be configured with various settings to tune the behavior.

During HTTP health checking Envoy will send an HTTP request to the upstream Endpoints.
It expects a 200 response if the host is healthy.
The upstream host can return 503 if it wants to immediately notify Envoy to no longer forward traffic to it.
It is important to note that these are health checks which Envoy implements and are separate from any other system such as those that exist in Kubernetes.

```yaml
# httpproxy-health-checks.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: health-check
  namespace: default
spec:
  virtualhost:
    fqdn: health.bar.com
  routes:
    - services:
        - name: s1-health
          port: 80
          healthCheck:
            path: /healthy
            intervalSeconds: 5
            timeoutSeconds: 2
            unhealthyThresholdCount: 3
            healthyThresholdCount: 5
        - name: s2-health # no health-check defined for this service
          port: 80
```

Health check configuration parameters:

- `path`: HTTP endpoint used to perform health checks on upstream service (e.g. `/healthz`). It expects a 200 response if the host is healthy. The upstream host can return 503 if it wants to immediately notify downstream hosts to no longer forward traffic to it.
- `host`: The value of the host header in the HTTP health check request. If left empty (default value), the name "contour-envoy-healthcheck" will be used.
- `intervalSeconds`: The interval (seconds) between health checks. Defaults to 5 seconds if not set.
- `timeoutSeconds`: The time to wait (seconds) for a health check response. If the timeout is reached the health check attempt will be considered a failure. Defaults to 2 seconds if not set.
- `unhealthyThresholdCount`: The number of unhealthy health checks required before a host is marked unhealthy. Note that for http health checking if a host responds with 503 this threshold is ignored and the host is considered unhealthy immediately. Defaults to 3 if not defined.
- `healthyThresholdCount`: The number of healthy health checks required before a host is marked healthy. Note that during startup, only a single successful health check is required to mark a host healthy.

#### WebSocket Support

WebSocket support can be enabled on specific routes using the `enableWebsockets` field:

```yaml
# httpproxy-websockets.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: chat
  namespace: default
spec:
  virtualhost:
    fqdn: chat.example.com
  routes:
    - services:
        - name: chat-app
          port: 80
    - conditions:
      - prefix: /websocket
      enableWebsockets: true # Setting this to true enables websocket for all paths that match /websocket
      services:
        - name: chat-app
          port: 80
```

#### Prefix Rewrite Support

Indicates that during forwarding, the matched prefix (or path) should be swapped with this value.
This option allows application URLs to be rooted at a different path from those exposed at the reverse proxy layer.
The original path before rewrite will be placed into the into the `x-envoy-original-path` header.

```yaml
# httpproxy-prefix-rewrite.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: app
  namespace: default
spec:
  virtualhost:
    fqdn: app.example.com
  routes:
    - services:
        - name: app
          port: 80
    - conditions:
      - prefix: /service2
      prefixRewrite: "/" # Setting this rewrites the request from `/service2` to `/`
      services:
        - name: app-service
          port: 80
```

#### Permit Insecure

A HTTPProxy can be configured to permit insecure requests to specific Routes.
In this example, any request to `foo2.bar.com/blog` will not receive a 301 redirect to HTTPS, but the `/` route will:

```yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: tls-example-insecure
  namespace: default
spec:
  virtualhost:
    fqdn: foo2.bar.com
    tls:
      secretName: testsecret
  routes:
    - services:
        - name: s1
          port: 80
    - conditions:
      - prefix: /blog
      permitInsecure: true
      services:
        - name: s2
          port: 80
```

#### ExternalName

HTTPProxy supports routing traffic to service types `ExternalName`.
Contour looks at the `spec.externalName` field of the service and configures the route to use that DNS name instead of utilizing EDS.

There's nothing specific in the HTTPProxy object that needs to be configured other than referencing a service of type `ExternalName`.

NOTE: The ports are required to be specified.

```yaml
# httpproxy-externalname.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    run: externaldns
  name: externaldns
  namespace: default
spec:
  externalName: foo-basic.bar.com
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  type: ExternalName
```

## HTTPProxy inclusion

TODO(youngnick) document inclusion

### Orphaned HTTPProxy children

It is possible for HTTPProxy objects to exist that have not been delegated to by another HTTPProxy.
These objects are considered "orphaned" and will be ignored by Contour in determining ingress configuration.

### Restricted root namespaces

HTTPProxy inclusion allows for Administrators to limit which users/namespaces may configure routes for a given domain, but it does not restrict where root HTTPProxy may be created.
Contour has an enforcing mode which accepts a list of namespaces where root HTTPProxy are valid.
Only users permitted to operate in those namespaces can therefore create HTTPProxy with the `virtualhost` field.

This restricted mode is enabled in Contour by specifying a command line flag, `--root-namespaces`, which will restrict Contour to only searching the defined namespaces for root HTTPProxy. This CLI flag accepts a comma separated list of namespaces where HTTPProxy are valid (e.g. `--root-namespaces=default,kube-system,my-admin-namespace`).

HTTPProxy with a defined `virtualhost` field that are not in one of the allowed root namespaces will be flagged as `invalid` and will be ignored by Contour.

Additionally, when defined, Contour will only watch for Kubernetes secrets in these namespaces ignoring changes in all other namespaces.
Proper RBAC rules should also be created to restrict what namespaces Contour has access matching the namespaces passed to the command line flag.
An example of this is included in the [examples directory](../examples/root-rbac) and shows how you might create a namespace called `root-httproxies`.

> **NOTE: The restricted root namespace feature is only supported for HTTPProxy CRDs.
> `--root-namespaces` does not affect the operation of `v1beta1.Ingress` objects**

## TCP Proxying

HTTPProxy supports proxying of TLS encapsulated TCP sessions.

_Note_: The TCP session must be encrypted with TLS.
This is necessary so that Envoy can use SNI to route the incoming request to the correct service.

### TLS Termination at the edge

If `spec.virtualhost.tls.secretName` is present then that secret will be used to decrypt the TCP traffic at the edge.

```yaml
# httpproxy-tls-termination.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: example
  namespace: default
spec:
  virtualhost:
    fqdn: tcp.example.com
    tls:
      secretName: secret
  tcpproxy:
    services:
    - name: tcpservice
      port: 8080
    - name: otherservice
      port: 9999
      weight: 20
```

The `spec.tcpproxy` key indicates that this _root_ HTTPProxy will forward the de-encrypted TCP traffic to the backend service.

### TLS passthrough to the backend service

If you wish to handle the TLS handshake at the backend service set `spec.virtualhost.tls.passthrough: true` indicates that once SNI demuxing is performed, the encrypted connection will be forwarded to the backend service.
The backend service is expected to have a key which matches the SNI header received at the edge, and be capable of completing the TLS handshake. This is called SSL/TLS Passthrough.

```yaml
# httpproxy-tls-passthrough.yaml
apiVersion: projectcontour.io/v1alpha1
kind: HTTPProxy
metadata:
  name: example
  namespace: default
spec:
  virtualhost:
    fqdn: tcp.example.com
    tls:
      passthrough: true
  tcpproxy:
    services:
    - name: tcpservice
      port: 8080
    - name: otherservice
      port: 9999
      weight: 20
```

## Status Reporting

There are many misconfigurations that could cause an HTTPProxy or delegation to be invalid.
To aid users in resolving these issues, Contour updates a `status` field in all HTTPProxy objects.
In the current specification, invalid HTTPProxy are ignored by Contour and will not be used in the ingress routing configuration.

If an HTTPProxy object is valid, it will have a status property that looks like this:

```yaml
status:
  currentStatus: valid
  description: valid HTTPProxy
```

If the HTTPProxy is invalid, the `currentStatus` field will be `invalid` and the `description` field will provide a description of the issue.

As an example, if an HTTPProxy object has specified a negative value for weighting, the HTTPProxy status will be:

```yaml
status:
  currentStatus: invalid
  description: "route '/foo': service 'home': weight must be greater than or equal to zero"
```

Some examples of invalid configurations that Contour provides statuses for:

- Negative weight provided in the route definition.
- Invalid port number provided for service.
- Prefix in parent does not match route in delegated route.
- Root HTTPProxy created in a namespace other than the allowed root namespaces.
- A given Route of an HTTPProxy both delegates to another HTTPProxy and has a list of services.
- Orphaned route.
- Delegation chain produces a cycle.
- Root HTTPProxy does not specify fqdn.