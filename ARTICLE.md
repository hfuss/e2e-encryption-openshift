## Easier End-to-End TLS for Spring Boot Apps using OpenShift
**Hayden Fuss**  
**Rough Draft**

### Introduction

One of the greatest features of OpenShift is its abstractions
for exposing, securing, and properly load-balancing your web
apps or services.

The goal of this series is to cover some self-service features a developer
could leverage to automate end-to-end TLS for their web apps, particularly
Spring Boot, as opposed to the more commonly used edge terminated TLS.

Service meshes like [Istio](https://istio.io/) are certainly making end-to-end
TLS (as well as [mTLS]()) even easier; however,
[OpenShift Service Mesh](https://docs.openshift.com/container-platform/3.11/servicemesh-install/servicemesh-install.html),
RedHat's enterprise flavor of Istio for OpenShift, will still be in
[tech preview](https://docs.openshift.com/container-platform/4.1/release_notes/ocp-4-1-release-notes.html#ocp-4-1-technology-preview)
for [OpenShift 4.1](https://docs.openshift.com/container-platform/4.1/welcome/index.html).
The approach we will cover is a decent alternative that can be entirely
automated without (much) help from a cluster administrator.

For these examples, we will assume our OpenShift cluster
administrators have taken care of two important details:

1. There is a `*.apps.example.com` DNS record pointed
   to the cluster's load-balancer.
2. The cluster's load-balancer has a valid TLS certificate for
   `*.apps.example.com`.

If this is not the case for your cluster(s), work with your cluster
administrators in getting a similar setup made and have them consider installing
add-ons such as
[External DNS](https://github.com/kubernetes-incubator/external-dns) and
[Certificate Manager](https://github.com/jetstack/cert-manager).

The requirements for us are as follows:

* Self-service, automated end-to-end TLS for Spring Boot apps, whether it be
  external or internal traffic
* Avoid managing external certificates directly, preserving a separation of
  duties between developers and cluster administrators
* Path-based routing for external traffic

### Part One: Re-encrypt TLS and Automated Internal Certificates with OpenShift

#### Introducing the `Route`

Before
[`Ingress`](https://kubernetes.io/docs/concepts/services-networking/ingress/)
was added in Kubernetes in 1.1, the OpenShift developers had provided
[`Route`'s](https://docs.openshift.com/container-platform/3.11/architecture/networking/routes.html).
Both make managing external access to HTTP/HTTPS `Service`'s running on a
cluster relatively easy. However, there are some notable differences,
see the [Appendix](#ingress-vs-route) for a more in-depth
comparison between `Route`'s and `Ingress`'s if you are
interested.

Overall, a `Route` is a slightly simpler abstraction which provides
more robust support for handling TLS out-of-the-box.

#### HTTPS Made Easy

Using a `Route` we can easily expose an HTTP `Service`
with a hostname under a particular path:

```
---
kind: Service
apiVersion: v1
metadata:
  name: my-component-v1
  namespace: my-app
  labels:
    app: my-app
    component: my-component
    version: v1
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: my-app
    component: my-component
    version: v1
---
kind: Route
apiVersion: v1
metadata:
  name: my-route
  namespace: my-app
  labels:
    app: my-app
    component: my-component
spec:
  host: my-component.apps.example.com
  path: /api/v1
  to:
    name: my-component-v1
    kind: Service
  port:
    targetPort: http
```

We can also externally secure our web service with edged-terminated TLS and
ensure our users are always redirected to the HTTPS port of the router:

```
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

Now our `Route` as a whole looks like:

```
---
kind: Route
apiVersion: v1
metadata:
  name: my-route
  namespace: my-app
  labels:
    app: my-app
    component: my-component
spec:
  host: my-component.apps.example.com
  path: /api/v1
  to:
    name: my-component-v1
    kind: Service
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

#### End-to-End TLS Made Easier

We need our `Route` to no longer edge-terminate TLS,
which leaves us with two options for the termination policy: `passthrough` and
`reencrypt`.

In short, `reencrypt` is better for our needs because it still supports
path-based routing, and does not require us to obtain a certificate issued for
the external hostname. For a more in-depth overview of `reencrypt` versus
`passthrough` check out this Red Hat
[blog post](https://blog.openshift.com/self-serviced-end-to-end-encryption-approaches-for-applications-deployed-in-openshift/).

With `reencrypt` termination, our app will need to serve a certificate that
our router trusts and ideally is valid for our `Service`'s hostname.
Luckily, OpenShift has a special `Service` annotation]() that will provide
the internal certificate we need, all you add is the following:

```
annotations:
    service.alpha.openshift.io/serving-cert-secret-name: my-component-v1-tls
```

OpenShift will then make a [`kubernetes.io/tls`]() type
`Secret` called `my-component-v1`, and it will contain the key
and cert in PEM format. We will discuss ways you can have your
Spring Boot application consume this `Secret` in the next article.

Putting this altogether our setup now looks like:

```
---
kind: Service
apiVersion: v1
metadata:
  name: my-component-v1
  namespace: my-app
  labels:
    app: my-app
    component: my-component
    version: v1
  annotations:
    service.alpha.openshift.io/serving-cert-secret-name: my-component-v1-tls
spec:
  type: ClusterIP
  ports:
    - port: 443
      targetPort: https
      protocol: TCP
      name: https
  selector:
    app: my-app
    component: my-component
    version: v1
---
kind: Route
apiVersion: v1
metadata:
  name: my-route
  namespace: my-app
  labels:
    app: my-app
    component: my-component
spec:
  host: my-component.apps.example.com
  to:
    name: my-component-v1
    kind: Service
  port:
    targetPort: https
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: Redirect
```

At last we have everything in place at the infrastructure level
for end-to-end TLS!

#### Next Steps

So far so good, but again we had made some
assumptions regarding DNS and the cluster router. If
your cluster's router does not have the TLS certificate for
your `Route`'s hostname you can also provide the certificate and key within
your `Route`'s manifest, see the [documentation]() for more details.

In the next article, we are going to bootstrap TLS and configure the Java keystore
for our Spring Boot apps, and then we will begin automating this setup by
writing a generic Helm chart.

#### Appendix

##### `Ingress` vs. `Route`

`Ingress`'s have the following features:
- Configures a proxy or load-balancer, such as NGINX, providing a single, external IP address for you to use. Depending on the underlying cloud platform or infrastructure you are using, your ingress controller will handle this configuration.
- Defines path-based routing to fan out to multiple `Service`'s for mutliple paths whether it be for a certain hostname(s) or any hostname
- A single `Ingress` can also define routing rules for multiple hostnames using name-based virtual hosting
- TLS `Secret`'s can be provided to the `Ingress` to configure edge terminated HTTPS
- Simple load-balancing schemes such as round robin and weight-based

On the other hand, `Route`'s provide the following:

- Leverages the existing HAProxy, F5 Big IP, or NGINX router that has been configured for the OpenShift cluster. As a result, the IP address is often the same for all `Route`'s and name-based virtual hosting is always used.
- A `Route` can only be defined for a single hostname and optional path. Multiple `Route`'s are required to configure different paths under the same hostname.
- Simple weight-based load-balancing for multiple backend `Service`'s
- TLS certificates and keys can be provided. However, a `Route` can serve the existing certs and keys that are configured on the cluster's load balancer.
- TLS can either be edge-terminated, passed through, or re-encrypted. For re-encrypt, a CA certificate can be provided so that the router trusts the destination.
- Insecure HTTP traffic can be automatically redirected to the load-balancer's HTTPS port

thing

### Part Two: Bootstrapping End-to-end TLS for Spring Boot Apps on OpenShift

TODO: recap from Part One

#### How to HTTPS with Spring Boot

Show `application.yml` snippets that are required. Point out the caveat of needing
a keystore. Explain some of the drawbacks of keystores i.e. reloading certs.

#### Creating PKCS12 Keystores with OpenSSL

Walking through the simple Bash with `openssl` that can take a set of PEMs
and make a generic keystore.

#### Bootstrapping Spring Boot with HTTPS via Init Containers

Combine the `application.yml` with the `keystore.sh`, and tie it all together
with Kubernetes and the special Service.

### Part Three: Handling Certificate Expiration for Spring Boot Apps

TODO
