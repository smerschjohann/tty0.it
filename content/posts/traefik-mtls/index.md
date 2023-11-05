---
title: "Securing Services with Traefik"
summary: "An Overview of Authentication Strategies"
date: 2023-11-03
weight: 1
chapter: false
draft: true
---

First, what is Traefik?

Traefik is a modern Reverse Proxy and Load Balancer that can be easily integrated into Kubernetes, any container engine or even on bare metal. It is the default Ingress Controller for k3s but can be installed on any Kubernetes system. With Traefik, you can define resources as follows:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-service
  namespace: test
spec:
  ingressClassName: traefik
  rules:
  - host: '*.domain.tld'
    http:
      paths:
      - backend:
          service:
            name: my-service
            port:
              number: 8080
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - '*.domain.tld'
    secretName: wildcard-domain-tld-tls
```

The Traefik Controller will monitor the cluster for changes on this resource and update its routing configuration accordingly. Ultimately, the service **my-service** in namespace **test** will be accessible on all domains matching `*.domain.tld`, such as _code.domain.tld_.

This means that Traefik acts as your gateway to services you wish to expose over HTTPS. Of course, this also comes with risks. What if you want to restrict access to a specific group, such as your family or colleagues?

The general solution lies in _Authentication_ and _Authorization_. You need to identify _who_ is attempting to access your service before you can determine _what_ they are permitted to do.

So what options do we have for authentication?

## Device-Level Authentication

The simplest method is IP whitelists. While this will only work with confidence on well protected networks (not the internet), it might be sufficient for certain services.

Bear in mind that you must have complete control over the network to depend on this method. On the Internet, a third party could potentially spoof IP addresses and bypass this security measure.

Only on private networks where you have control over the infrastructure including managed switches secured by authentication protocols like 802.1x or by VPNs like Wireguard, you could feasibly identify a user by their IP address.
If that's not the case, it's crucial not to blindly trust IP addresses. But it can still add an additional layer of protection.

You can implement this type of authentication using the [IP Whitelisting Middleware](https://doc.traefik.io/traefik/middlewares/http/ipwhitelist/).

The Middleware can then be activated by annotating the `Ingress` resource.

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: ipwhitelist
  namespace: test
spec:
  ipWhiteList:
    sourceRange:
      - 192.168.1.0/24
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-service
  namespace: test
  annotations:
    # convention: <namespace>-<middleware name>@kubernetescrd
    traefik.ingress.kubernetes.io/router.middlewares: test-ipwhitelist@kubernetescrd
```


## Identity Management-Based Authentication

A more common approach involves user credentials, such as a username and password, ideally coupled with Multi-Factor Authentication (MFA). These credentials can be verified locally or through LDAP, SAML, or any other Identity Management System.

When combined with protocols like OAuth2 or OIDC, this can be seamlessly integrated into a service. The service can then rely on a signed token to authenticate the user appropriately.

However, this method has its drawbacks: Each application must validate the tokens or outsource this to an OAuth2 Reverse Proxy. This is feasible for browser-based applications but can be problematic for CLI-based tools, which may not support such authentication methods or are limited to specific authentication flows.

This authentication variant can be implemented individually for each service or by using the [ForwardAuth Middleware](https://doc.traefik.io/traefik/middlewares/http/forwardauth/) in combination with tools like [OAuth2-Proxy](https://oauth2-proxy.github.io/oauth2-proxy/docs/features/endpoints). Please refer to the [endpoint documentation](https://oauth2-proxy.github.io/oauth2-proxy/docs/features/endpoints) and in particular the `/oauth2/auth` endpoint.

I leave implementations details to the reader. If you like to contribute a variant, feel free to contact me.


## TLS based

Another way to protect your sites: Mutual TLS.

While it is lesser known, it is one of the most secure variant.

So how does it work? 