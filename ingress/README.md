# Ingress
- Make your HTTP (or HTTPS) network service available using a protocol aware configuration mechanism, that understands web concepts like URLs, hostnames, paths and more. The Ingress concept lets you map traffic to different backends based on rules you define via the Kubernetes API.
- Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defines on the Ingress resource.
- The below is the simple example where an ingress sends all the traffic to one service.
![alt ingress](images/ingress.svg)
- An Ingress may be configures to give services externally-reachable URLs, load balence traffic, terminate SSL/TLS, and offer name-based virtual hosting.
- An `Ingress Controller` is responsible for fulfilling the ingress, usually with a load balancer, though it may also configure your edge router or addditional frontends to help handle the traffic.
- An Ingress does not expose arbitrary ports or protocols. Exposing services other than HTTP and HTTPS to the internet typically uses a service of type `NodePort` or `LoadBalencer`

# Prerequisites
- You must have an Ingress controller to satisfy an Ingress. Only creating an Ingress resource has no effect.
- You may need to deploy an Ingress Controler such as `ingress-nginx`. You can choose from a number of [ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

# The Ingress Resource
- The following configuration shows a minimal ingress resource example
    ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: minimal-ingress
      namespace: app
    spec:
      ingressClassName: nginx-example
      rules:
        - http:
            paths:
              - path: /test
                pathType: Prefix
                backend:
                  service:
                    name: test-service
                    port:
                      number: 80             
    ```
- An ingress needs `apiVersion`, `kind`, `metadata` and `spec` fields. The name of the ingress must be a valid DNS subdomain name.
- The ingress spec has all the information needed to configure a load balancer or a proxy server. Most importantly, it contains a list of rules matched against all incoming requests. Ingress resource only supports rules for directing HTTP(S) traffic.
- If the `ingressClassName` is omitted, a default ingress class should be defined.

## Ingress Rules
- Each HTTP rule contains the following information:
  - An optional host. In this example, no host is specified, so the rule applies to all inbound HTTP traffic through the IP Address specified. If a host is provided (foo.bar.com), the rules apply to that host.
  - A list of paths (/test), each of which has an associated backend defined with a `service.name` and a `service.port.name` or `service.port.number`. Both the host and path must match the content of an incoming request before the load balancer directs traffic to the referenced service.
  - A backend is a combination os Service and port names as described in the Service doc or a custom resource backend by way of a CRD. HTTP (and HTTPS) requests to the ingress that match the host and path of the rule are sent to the listed backend.
- A `defaultBackend` is often configured in an ingress controller to service any requests that do not match a path on the spec.

## Default Backend
- An Ingress with no rules sends all the traffic to a single default backend and `.spec.defaultBackend` is the backend that should handle requests in that case.
- The defaultBackend is conventionally a configuration option of the Ingress Controller and is not specified in your Ingress resources.
- If no `.spec.rules` are specified, `.spec.defaultBackend` must be specified. if defaultBackend is not set, the handling of requests that do not match any of the rules will be up to the ingress controller.
- If none of the hosts or paths match the HTTP requests in the Ingress objects, the traffic is routed to your default backend.

## Resource Backends
- A `Resource` backend is an ObjectRef to another Kubernetes resource within the same namespace as the Ingress object. A `Resource` is mutually exclusive setting with a Service and will fail validation if both are specified.
- A common usage for a `Resource` backend is to ingress data to an object storage backend with static assets.
    ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: resource-backend-ingress
    spec:
      defaultBackend:
        resource:
          kind: StorageBucket
          name: static-assets
          apiGroup: k8s.example.com
          
      rules:
        - http:
            - paths:
                - path: /icons
                  pathType: ImplementationSpecific
                  backend:
                    resource:
                      apiGroup: k8s.example.com
                      kind: StorageBucket
                      name: icon-assets
    ```

## Path Types
- Each path in an Ingress is required to have a corresponding path type. Paths that do not include an explicit `pathType` will fail validation. There are three supported path types:
  - `ImplementationSpecific`: With this path type, matching is up to the IngressClass. Implementations can treat this as a separate pathType or treat it identically to `Prefix` or `Exact` path types.
  - `Exact`: Matches the URL path exactly and with case sensitivity.
  - `Prefix`: Matches based on a URL path prefix split by `/`. Matching is case-sensitive and done on a path element by element basis. A path element refers to the list of labels in the path split by the `/` separator.

## Hostname wildcards
- Hosts can be precise matches (for ex: `foo.bar.com`) or a wildcard (for example `*.foo.com`). Precise matches require that the HTTP `host` header matches the `host` field. Wildcard matches require the HTTP `host` header is equal to the suffix of the wildcard rule.

    -------------------------------
    Host | Host Header | Match?
    ----- | ------------ | --------
    *.foo.com | bar.foo.com | Matches based on shared suffix |
    *.foo.com | baz.bar.foo.com | No match, wildcard only covers a single DNS label |
    *.foo.com | foo.com | No match, wildcard only covers a single DNS label |

    ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: wildcard-host-ingress
    spec:
      rules:
        - host: "foo.bar.com"
          http:
            paths:
              - path: "/bar"
                pathType: Prefix
                backend:
                  service:
                    name: service-1
                    port:
                      number: 80
        - host: "*.foo.com"
          http:
            paths:
              - path: "/foo"
                pathType: Prefix
                backend:
                  service:
                    name: service-2
                    port:
                      number: 80
    ```
## Ingress Class
- Ingresses can be implemented by different controllers, often with different configuration.
- Each ingress should specify a class, a reference to an IngressClass resource that contains additional configuration including the name of the controller that should implement the class.
  
    ```
    apiVersion: networking.k8s.io/v1
    kind: IngressClass
    metadata:
        name: external-lb
    spec:
        controller: example.com/ingress-controll
        parameters:
            apiGroup: k8s.example.com
            kind: IngressParameters
            name: external-lb
    ```
- The `.spec.parameters` field of an IngressClass lets you reference another resource that provides configuration related to that IngressClass.
- The specific type of parameters to use depends on the ingress controller that you specify in the `.spec.controller` filed in the IngressClass.

## Types of Ingress
### 1. Ingress backed by a single service
- There are existing kubernetes concepts that allow you to expose a single Service. You can also do this with an Ingress by specifying a default backend with no rules.

    ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
        name: test-ingress
    spec:
        defaultBackend:
            service:
                name: test-service
                port:
                    number: 80
    ```

### 2. Simple fanout
- A fanout configuration rotes traffic from a single IP address to more than one Service, based on the HTTP URI being requested. An Ingress allows you to keep the number of load balancers down to a minimum.
- For example, the setup as follows

![alt ingress fanout](images/ingressFanOut.svg)
- For the above example, the Ingress configuration will looks like

  ```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-fanout
  spec:
    rules:
      - host: foo.bar.com
        http:
          paths:
            - path: /foo
              pathType: Prefix
              backend:
                service:
                  name: foo-service
                  port:
                    number: 80
            - path: /bar
              pathType: Prefix
              backend:
                service:
                  name: bar-service
                  port:
                    number: 80
  ```

### 3. Name based virtual hosting
- Name-based virtual hosts support routing HTTP traffic to multiple host names at the same IP address

![alt name based virtual hosting](images/ingressNameBased.svg)
- The following ingress tells the backing load balancer to route requests based on the Host Header. 
  ```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-name-based
  spec:
    rules:
      - host: foo.bar.com
        http:
          paths:
            - path: "/"
              pathType: Prefix
              backend:
                service:
                  name: service1
                  port:
                    number: 80
      - host: bar.foo.com
        http:
          paths:
            - path: "/"
              pathType: Prefix
              backend:
                service:
                  name: service2
                  port:
                    number: 80
  ```
- If you create an ingress resource without any hosts defined in the rules, then any web traffic to the IP address of your ingress controller can be matched without a name based virtual host being required.
- For example, an Ingress routes traffic requested for `first.bar.com` to `service` and `second.bar.com` to `service2`, and any traffic whose request host header doesn't match `first.bar.com` and `second.bar.com` to `service3`

### 4. TLS
- You can secure a Ingress by specifying a Secret that contains a TLS private key and certificate. The ingress resource only supports a single TLS port, 443, and assumes TLS termination at the ingress point (traffic to the Service and its Pods is in plaintext).
- The TLS Secret must contain keys names `tls.cert` and `tls.key` that contain the certificate and private key to use for TLS.
  ```
  apiVersion: v1
  kind: Secret
  metadata:
    name: ingress-tls-secret
    namespace: default
  type: kubernetes.io/tls
  data:
    tls.cert: base64 encoded cert
    tls.key: base64 encoded key
  ```
- Referencing the secret in the ingress tells the ingress controller to secure the channel from the client to the load balancer using TLS.
- Note: TLS will not work on the default rule because the certificates would have to be issued for all the possible subdomains. Therefore, the hosts in the TLS section need to explicitly match the host in the rules section.
  
  ```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-tls
  spec:
    tls:
      - hosts:
          - bar.foo.com
        secretName: ingress-tls-secret
    rules:
      - host: bar.foo.com
        http:
          paths:
            - path: "/"
              pathType: Prefix
              backend:
                service:
                  name: service1
                  port:
                    number: 80
  ```