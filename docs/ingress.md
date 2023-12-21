# Ingress

*Adapted from: https://kubernetes.io/docs/concepts/services-networking/ingress/*

## Terminology

* `Node` - A worker machine in Kubernetes (k8s). Part of a cluster.
* `Cluster` - A set of Nodes that run containerised apps managed by k8s.
* `Edge Router` - A router that enforces firewall policy on your cluster. This could be a gateway managed by a cloud provider or a physical piece of hardware.
* `Cluster Network` - A set of links, logical or physical, that facilitate communication within a cluster according to the k8s netwroking model.
* `Service` - A k8s service that identifies a set of Pods using label selectors. Unless mentioned otherwise, Services are assumed to have virtual IPs only routable within the cluster network.

## What is Ingress?

Ingress exposes HTTP(S) routes from outside to cluster to services within the cluster. Traffic routing is controlled by routes defined on the Ingress resource.

Below is a simple example of where an ingress sends all its traffic to one service: 

[DIAGRAM GOES HERE]

An Ingress may be configured to give services externally-reachable URLs, load balance traffic, terminate SSL / TLS, and offer name-based virtual hosting. An Ingress Controller is responsible for fulfilling the Ingress, usually with a load-balancer, though it may also configure your edge router or additional frontends to help handle the traffic.

An Ingress does not expose arbitrary ports or protocols. Exposing services other than HTTP(S) to the internet typically uses a service of type `Service.Type=NodePort` or `Service.Type=LoadBalancer`

## Pre-requisites

You must have an Ingress Controller to satisfy an Ingress (We use [Kong](https://github.com/Kong/kubernetes-ingress-controller#readme)). If you only create an Ingress resource, it'll have no effect. Ideally all ingress controllers would fit the reference k8s ingress specs, but in reality there can be nuances. Check their docs to be sure.

## The Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: minimal-ingress
    annotations: 
        nginx.ingress.kubernetes.io/rewrite-target: /
spec:
    ingressClassName: nginx-example
    rules:
    - http:
        paths:
        - path: /testpath
          pathType: Prefix
          backend:
            service:
                name: test
                port:
                    number: 80 
```

An Ingress needs `apiVersion`, `kind`, `metadata` and `spec` fields. The name of an Ingress object must be a valid DNS sub-domain name. Ingress frequently uses anootations to configure some options depending on the Ingress controller, e.g. the rewrite-target annotation. Different ingress controllers support different annotations so review the docs of your chosen controller to see what's supported.

The Ingress `.spec` has all the information needed to configure a load balancer or proxy server. Most importantly, it contains a list of rules matched against all incoming requests. Ingress only supports rules for directing HTTP(S) traffic.

If the `IngressClassName` is omitted, a default Ingress class should be defined.

## Ingress Rules
Each HTTP rule contains the following information:

* An optional host. In this example, no host is specified, so the rule applies to all inbound HTTP traffic through the IP address specified. If a host is provided (i.e. foo.bar.com), the rules apply to that host.
* A list of paths (for example /testpath), each of which has an associated backend defined with a `serivice.name` and a `service.port.name` or `service.port.number`. Both the host and path must match the content of an incoming request before the load balancer directs traffic to the referenced service.
* A backend is a combinnation of Service and port names as described in the Service doc or a custom resource backend by way of a CRD. HTTP(S) requests to the Ingress that matches the host and path of the rule are sent to the listed backend.

A `defaultBackend` is often configured in an Ingress controller to service any requests that do not match a path in the spec.

## DefaultBackend
An Ingress with no rules sends all traffic to a single default backend and `.spec.defaultBackend` is the backend that should handle requests in that case, the `defaultBackend` is conventially a configuration option of the Ingress controller (i.e. Kong) and is not specified in your Ingress resources. If no `.spec.rules` are specified, `.spec.defaultBackend` must be specified. If `defaultBackend` is not set, the handling of requests that do not match any of the rules will be up to the ingress controller.

If none of the hosts or paths match the HTTP request in the Ingress objects, the traffic is routed to your default backend.

## Resource backends
A `Resource` backend is an ObjectRef to another Kubernetes resource within the same namespace as the Ingress object. A `Resource` is a mutually exclusive setting with Service, and will fail validation if both are specified. A common usage for a `Resource` backend is to ingress data to an object storage backend with static assets.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: ingress-resource-backend
spec:
    defaultBackend:
        resource:
            apiGroup: k8s.example.com
            kind: StorageBucket
            name: static-assets
    rules:
        - http:
            paths:
                - path: /icons
                pathType: ImplementationSpecific
                backend:
                    resource:
                        apiGroup: k8s.example.com
                        kind: StorageBucket
                        name: icon-assets
```

After creating the above Ingress, you can view it with the following command:

```bash
kubectl describe ingress ingress-resource-backend
```

```bash
Name:             ingress-resource-backend
Namespace:        default
Address:
Default backend:  APIGroup: k8s.example.com, Kind: StorageBucket, Name: static-assets
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /icons   APIGroup: k8s.example.com, Kind: StorageBucket, Name: icon-assets
Annotations:  <none>
Events:       <none>
```

## Path Types
Each path of an Ingress is required to have a coresponding path type. Paths that do not include an explicit pathType will fail validation. There are three supported path types:

* ImplementationSpecific: With this path type, matching is up to the IngressClass. Implementations can treat this as a seperate `pathType` or treat it identically to `Prefix` or `Exact` path types.
* `Exact`: Matches the URL path exactly and with case sensitivity.
* `Prefix`: Macthes based on a URL path prefix split by `/`. Matching is case sensitive and done on a path element by element basis. A path element refers to a list of labels in the path split by the `/` operator. A request is a match for path *p* if every *p* is an element-wise prefix of *p* of the request path.

!!! Note
        If the last element of the path is a substring of the last element in the request path, it is a not a match (e.g. /foo/bar matches /foo/bar/baz but not foo/barbaz)

#### Multiple matches
In some cases, multiple paths within in an Ingress will match a request. In those cases, precedence will be given first to the longest matching path. if two paths are still equally matched, precendence will be given to paths with an exact path type over a prefix path type.

## Hostname wildcards
Hosts can be precise matches (e.g. "`foo.bar.com`") or a wildcard ("`*.foo.com`"). Precise matches require that the HTTP `host` header matches the `host` field. Wildcard matches require the HTTP `host` header is equal to the suffix of the wildcard rule.

| Host      | Host header     | Match?                                            |
| --------- | --------------- | ------------------------------------------------- |
| *.foo.com | bar.foo.com     | Matches based on shared suffix                    |
| *.foo.com | baz.bar.foo.com | No match, wildcard only covers a single DNS label |
| *.foo.com | foo.com         | No match, wildcard only covers a single DNS label |

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 80
```
## Ingress Class
Ingresses can be implemented by different controllers, often with different configuration. Each ingress should specify a class, a reference to an IngressClass resource that contains additional configuration including the name of the controller that should implement the class.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata: 
    name: external-lb
spec: 
    controller: example.com/ingress-controller
    parameters:
        apiGroup: k8s.example.com
        kind: IngressParameters
        name: external-lb
```

The `.spec.parameters` field of an IngressClass lets you reference another resource that provides configuratio related to that IngressClass.

The specific type of parameters to use depends on the ingress controller that you specify in the `.spec.controller` field of the IngressClass.

### IngressClass scope
Depending on your ingress controller, you may be able to use parameters that you set cluster-wide or just for one namespace.

The default scope for IngressClass parameters is cluster-wide.

If you set the `.spec.parameters` field and don't set `.spec.parameters.scope`, or if you set `.spec.parameters.scope` to `Cluster`, then the IngressClass refers to a cluster-scoped resource. The `kind` (in combination with the `apiGroup`) of the parameters refers to a cluster-scoped API (possibly a custom resource), and the `name` of the parameters identifies a specific cluster scoped resource for that API.

For example:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
    name: external-lb-1
spec:
    controller: example.com/ingress-controller
    parameters:
        # The parameters for this IngressClass are specified in a
        # ClusterIngressParameter (API group k8s.example.net) named
        # "external-config-1". This definition tells Kubernetes to
        # look for a cluster-scoped parameter resource.
        scope: Cluster
        apiGroup: k8s.example.net
        kind: ClusterIngressParameter
        name: external-config-1

```

### Deprecated annotation
Before the IngressClass resource and `ingressClassName` field were added in k8s 1.18, Ingress classes were specified with a `kubernetes.io/ingress.class` annotation on the Ingress. This anootation was never formally defined, but was widely supported by Ingress controllers.

The newer `ingressClassName` field on Ingresses is a replacement for that annotation, but it is not a direct equivalent. While the annotation was generally used to reference the name of the Ingress controller that should implement the Ingress, the field is a reference to an IngressClass resource that contains additional Ingress configuration, including the name of the Ingress controller.

### Default IngressClass
You can mark a particular IngressClass as default for your cluster. Setting the `ingressclass.kubernetes.io/is-default-class` annotation to `true` on an IngressClass resource will ensure that new Ingresses without an `ingressClassName` will be assigned this default IngressClass.

!!! Caution
        If you have more than one IngressClass marked as the default for your cluster, the admission controller prevents creating new Ingress objects that don't have an `ingressClassName` specified. You can resolve this by ensuring that at most 1 IngressClass is marked as default in your cluster.

There are some ingress controllers, that work without the definition of a default `IngressClass`. For example, the Ingress-NGINX controller can be configured with a flag: `--watch-ingress-without-class`. It is recommended though to specify the default `IngressClass`

```yaml
apiVersion: networking,k8s.io/v1
kind: IngressClass
metadata:
    labels:
        app.kubernetes.io/component: controller
    name: nginx-example
    annotations:
        ingressclass.kubernetes.io/is-default-class: "true"
spec:
    controller: k8s.io/ingress-nginx
```

## Types of Ingress
### Ingress backed by a single Service
There are existing Kubernetes concepts that allow you to expose a single service. You can also do this with an Ingress by specifyng a *default backend* with no rules.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  defaultBackend:
    service:
      name: test
      port:
        number: 80
```

If you create it using `kubectl apply -f` you should be able to view the state of the Ingress you added:

```bash
kubectl get ingress test-ingress
```

```bash
NAME           CLASS         HOSTS   ADDRESS         PORTS   AGE
test-ingress   external-lb   *       203.0.113.123   80      59s
```

Where `203.0.113.123` is teh IP allocated by the Ingress controller to satisfy this Ingress.

!!! Note
        Ingress controllers and load balancers may rake a minute or two to allocate an IP address. Until that time, you often see te address listed as `<pending>`


### Simple fanout
A fanout configuration routes traffic from a single IP address to more than one Service, based on the HTTP URI being requested. An ingress allows you to keep the number of load balancers down to a minimum. As an example, a setup like the one in the diagram below:

[DIAGRAM]

Would require an Ingress configuration like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: ingress
metadata:
    name: simple-fanout-example
spec:
    rules:
    - host: foo.bar.com
      http:
        paths:
        - path: /foo
        pathType: Prefix
        backend:
            service: 
                name: service1
                port:
                    number: 4200
        - path: /bar
        pathType: Prefix
        backend: 
            service:
                name: service2
                port:
                    number: 8080
```

When you create the Ingress with `kubectl apply -f`:

```bash
kubectl describe ingress simple-fanout-example
```

```bash
Name:             simple-fanout-example
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:4200 (10.8.0.90:4200)
               /bar   service2:8080 (10.8.0.91:8080)
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     22s                loadbalancer-controller  default/test

```

The Ingress controller provisions an implementation-specific load balancer that satisfies the Ingress, as long as the Services (`service1`  and `service2`) exist. When it has done so, you can see the address of the load balancer in the Address field.

!!! Note
        Depending on the Ingress controller you are using, you may need to create a default http-backend Service

### Name-based virtual hosting
Name-based virtual hosts support routing HTTP traffic to multiple host names at the same IP address.

[DIAGRAM GOES HERE]

The following Ingress tells the backing load balancer to route requests based on the Host header.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: bar.foo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
```

If you create an Ingress resource without any hosts defined in the rules, then any web traffic to the IP address of your Ingress controller can be matched without a name based virtual host being required.

For example, the following Ingress routes traffic requested for `first.bar.com` to `service1`, `second.bar.com` to `service2`, and any traffic whose request host header doesn't match `first.bar.com` and `second.bar.com` to `service3`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress-no-third-host
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: second.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
  - http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service3
            port:
              number: 80
```

### TLS
You can secure an Ingress by specifying a Secret that contains a TLS private key and certificate. The ingress resource only supports a single TLS port, 443, and assumes TLS termination at the ingress point (traffic to the Service and its Pods is in plaintext). If the TLS configuration section in an Ingress specifies different hosts, they are multiplexed on the same port according to the hostname specified through the SNI (Server Name Indication) TLS extension (provided the Ingress controller supports SNI). The TLS secret mus contain keys named `tls.crt` and `tls.key` that contain the certificate and private key to use for TLS. For example:

```
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

Referencing this secret in an Ingress tells the Ingress controller to secure the channel from the client to the load balancer using TLS. You need to make sure TLS secret you created came from a certificate that contains a Common Name (CN), also known as a Fully Qualified Domain Name (FQDN) for `https-example.foo.com`.

!!! Note
        Keep in mind that TLS will not work on the default rule because the certificates would have to be issued for all the possible sub-domains. Therefore, `hosts` in the `tls` section need to expicitly match the `host` in the `rules` section.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
```

!!! Note
        There is a gap between TLS features supported by various Ingress controllers. Please refer to documentation on your platform's ingress controller to understand how TLS is configured in your environment.

### Load Balancing
An Ingress controller is bootstrapped with some load balancing policy settings that it applies to all Ingress, such as the load balancing algorithm, backend weight scheme, and others. More advanced load balancing concepts (e.g. persistent sessions dynamic weights) are not yet exposed through the Ingress. You can instead get these features through the load-balancer used for a Service.

It's also worth noting that even though health checks are not exposed directly through the Ingress, there are parallel concepts in k8s such as readiness probes that allow you to achieve the same end result. Please review the controller specific documentation to see how they handle healthchecks.








