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
Spring Boot applications. Lastly, we will cover how to handle
certificate expiration and the difficulties that arise with
Java keystores when doing so.

There will be links and code snippets throughout the articles,
but if you want something you could get your hands on check out
the [sample code](https://github.com/hfuss/e2e-encryption-openshift). For
these examples, we will assume our cluster
administrators have take care of two important details:

1. There is a `*.apps.example.com` DNS record pointed
to the cluster's load-balancer.
2. The cluster's load-balancer has a valid TLS certificate for `*.apps.example.com`.

If this is not the case for your cluster, work with your cluster administrators in getting this setup and have them consider tools such as [External DNS]() and [Certificate Manager]().

### Part One: Re-encrypt TLS and Automated Internal Certificates with OpenShift

#### Introducing the `Route`

So first things first, what are these great OpenShift
abstractions for exposing and load-balancing web services and
how do they differ from what else exists in the Kubernetes
landscape?

Before there was the `Ingress` in Kubernetes, the OpenShift developers had provided `Route`'s. Both make managing external access to HTTP/HTTPS `Service`'s running on a cluster relatively easy. However, there are some notable differences, see the [Appendix](#ingress-vs-route) for a more in-depth comparison between `Route`'s and `Ingress`'s if you are interested.

Overall, a `Route` in some ways is a slightly simpler
abstraction which provides more robust support for handling
TLS, and a greater separation of duties between the cluster
administrators and developers. Additionally, with OpenShift
3.10 there is now an ingress controller which will
automatically make a `Route` (or possibly multiple) in the
background for any `Ingress`'s you define.

#### HTTPS Made Easy

In pursuit of our goal of end-to-end TLS for our web services,
we will take advantage of what `Route`'s have to offer. Firstly, using a `Route` we can easily expose an HTTP `Service` with a hostname under a certain path:

```
---
kind: Service
apiVersion: v1
metadata:
  name: my-service-v1
  labels:
    app: my-app
    component: my-component
    version: v1
spec:

---
kind: Route
apiVersion: v1
metadata:
  name: my-route
  labels:
    app: my-app
    component: my-component
spec:

```

Even better, we can then secure our web service with edged
terminated TLS and the ensure our users are always redirected
to the HTTPS port:


```
---
kind: Route
apiVersion: v1
metadata:
  name: my-route
  labels:
    app: my-app
    component: my-component
spec:
```


#### End-to-End TLS Made Easier

`passthrough` vs `reencrypt`, why `passthrough` falls short.

How to then configure a `reencrypt` `Route`:

```

```

Autogenerating TLS certs for our internal `Service`'s.

```

```

#### Moving Forward

...

#### Appendix

##### `Ingress` vs. `Route`

`Ingress`'s have the following features:
- Configures a proxy or load-balancer, such as NGINX, providing a single, external IP address for you to use. Depending on the underlying cloud platform or infrastructure you are using, your ingress controller will handle this configuration
- Defines path-based routing to fan out to multiple `Service`'s for different paths whether it be for a certain hostname or any hostname
- A single `Ingress` can also define routing rules for multiple hostnames using name-based virtual hosting
- TLS `Secret`'s can be provided to the `Ingress` to configure edge terminated HTTPS
- Simple load-balancing schemes such as round robin and weight-based

On the other hand, `Route`'s provide the following:

- Leverages the existing HAProxy, F5 Big IP, or NGINX load balancer that has been configured for the OpenShift cluster. As a result, the IP address is often the same all `Route`'s and name-based virtual hosting is always used.
- A `Route` can only be defined for a single hostname and optional path. Multiple `Route`'s are required to configure different paths under the same hostname.
- Simple weight-based load-balancing for multiple backend `Service`'s
- TLS certificates and keys can be provided. However, a `Route` can serve the existing certs and keys that are configured on the cluster's load balancer providing a nice separation of duties between a sysadmin and a developer.
- TLS can either be edge-terminated, passed through, or re-encrypted. For re-encrypt, a CA certificate can be provided so that the router trusts the destination
- Insecure HTTP secure can be automatically redirected to

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
