## Easier End-to-End TLS with OpenShift
**Hayden Fuss**  
**Rough Draft**

### Introduction

One of the greatest features of OpenShift is its abstractions
for exposing, securing, and properly load-balancing your web
services.

This series is meant for platform engineers and application
developers who are new to OpenShift or looking to expand their
knowledge. The goal is to cover some useful features you
could leverage to implement end-to-end TLS for your web services
as opposed to the more commonly used edge terminated TLS.

We will then dive a little deeper to discuss some practices you
could use to automate this setup for your development teams, and
in particular we will cover what is involved in doing so for
Spring Boot applications. Lastly, we will go over how to handle
certificate expiration and the difficulties that arise with
Java keystores when doing so.

There will be links and code snippets throughout the articles,
but if you want something you could get your hands on check out
the [sample code](https://github.com/hfuss/e2e-encryption-openshift). For
these examples, we will assume our OpenShift cluster
administrators have taken care of two important details:

1. There is a `*.apps.example.com` DNS record pointed
to the cluster's load-balancer.
2. The cluster's load-balancer has a valid TLS certificate for `*.apps.example.com`.

If this is not the case for your cluster, work with your cluster administrators in getting a similar setup made and have them consider setting up tools such as [External DNS]() and [Certificate Manager]().

### Part One: Re-encrypt TLS and Automated Internal Certificates with OpenShift

#### Introducing the `Route`

So first things first, what are these great OpenShift
abstractions for exposing and load-balancing web services and
how do they differ from what else exists in the Kubernetes
landscape?

Before there was the `Ingress` in Kubernetes, the OpenShift
developers had provided `Route`'s. Both make managing external
access to HTTP/HTTPS `Service`'s running on a cluster
relatively easy. However, there are some notable differences,
see the [Appendix](#ingress-vs-route) for a more in-depth
comparison between `Route`'s and `Ingress`'s if you are
interested.

Overall, a `Route` in some ways is a slightly simpler
abstraction which provides more robust support for handling
TLS, and a greater separation of duties between the cluster
administrators and developers. Additionally, with OpenShift
3.10 there is now an ingress controller which will
automatically make a `Route` (or possibly multiple) in the
background for any `Ingress`'s you define.

#### HTTPS Made Easy

In pursuit of our goal of end-to-end TLS for our web services,
we will take advantage of what `Route`'s have to offer.
Firstly, using a `Route` we can easily expose an HTTP `Service`
with a hostname under a certain path:

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
  to:
    name: my-component-v1
    kind: Service
  port:
    targetPort: http
```

Even better, we can then secure our web service with edged
terminated TLS and the ensure our users are always redirected
to the HTTPS port of the load-balancer:

```
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

so now our `Route` as a whole looks like:

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

Alright so that was pretty easy and we are a few steps closer
to achieving end-to-end TLS. Moving forward, we first need our
`Route` to no longer edge terminate TLS, which leaves us with
two options for the termination policy: `passthrough` and
`reencrypt`.

With `passthrough`, the load-balancer will not participate in
the TLS handshake and will simply send all traffic straight to
the `Service` and then your `Pod`'s. That means your `Pod`'s
would need a valid TLS cert for `my-component.apps.example.com`
for the sake of the external client.

With `reencrypt`, the load-balancer will handle the TLS
handshake with the external client. Then it will do its own TLS
handshake with the `Pod`'s at the end of your `Service`,
re-encrypting the packets based on the TLS cert your pods are
serving. Depending on what cert your `Pod`'s serve, you can
specify a `destinationCACertificate` in the `Route` so that it
trusts your cert. Run `oc explain route.spec.tls` for more
details on all three policies and what additional options you
have.

Aside from needing a valid cert
`my-component.apps.example.com`, the other drawback to
`passthrough` is that internal `Service` hostname, i.e.
`my-component-v1.my-app.svc.cluster.local`, would still be
considered invalid. With `reencrypt` you could still achieve
this, but then your `Pod`'s need a valid cert for that internal
hostname... luckily OpenShift can automatically provide just
that!

OpenShift has a [special `Service` annotation]() that will tell
it to automatically create a cert, all you add is the following:

```
annotations:
    service.alpha.openshift.io/serving-cert-secret-name: my-component-v1-tls
```

OpenShift will then make a [`kubernetes.io/tls`]() type
`Secret` called `my-component-v1`, and it will contain the key
and cert in PEM format. We will discuss ways you can have your
application `Pod`'s consume this `Secret` in the next article.

But so, putting this altogether our setup now looks like:

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

And so at last we have everything in place at the routing level
for end-to-end TLS!

#### Moving Forward

So that was not terribly hard but again we made some
assumptions regarding DNS and the cluster load-balancer. If
your cluster's load-balancer does not have the TLS cert for
your `Route`'s hostname you can also provide the cert within
your `Route`'s manifest.

In the next article, we are going to automate this setup and
bootstrapping TLS for Spring Boot apps using a handy Helm chart.

#### Appendix

##### `Ingress` vs. `Route`

`Ingress`'s have the following features:
- Configures a proxy or load-balancer, such as NGINX, providing a single, external IP address for you to use. Depending on the underlying cloud platform or infrastructure you are using, your ingress controller will handle this configuration
- Defines path-based routing to fan out to multiple `Service`'s for mutliple paths whether it be for a certain hostname(s) or any hostname
- A single `Ingress` can also define routing rules for multiple hostnames using name-based virtual hosting
- TLS `Secret`'s can be provided to the `Ingress` to configure edge terminated HTTPS
- Simple load-balancing schemes such as round robin and weight-based

On the other hand, `Route`'s provide the following:

- Leverages the existing HAProxy, F5 Big IP, or NGINX load balancer that has been configured for the OpenShift cluster. As a result, the IP address is often the same for all `Route`'s and name-based virtual hosting is always used.
- A `Route` can only be defined for a single hostname and optional path. Multiple `Route`'s are required to configure different paths under the same hostname.
- Simple weight-based load-balancing for multiple backend `Service`'s
- TLS certificates and keys can be provided. However, a `Route` can serve the existing certs and keys that are configured on the cluster's load balancer.
- TLS can either be edge-terminated, passed through, or re-encrypted. For re-encrypt, a CA certificate can be provided so that the router trusts the destination.
- Insecure HTTP traffic can be automatically redirected to the load-balancer's HTTPS port

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
