---
title: "Securing Services with Traefik"
summary: "An Overview of Authentication Strategies"
date: 2023-11-06
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

The simplest method is a IP whitelist. While this will only work with confidence on well protected networks (not the internet), it might be sufficient for certain services.

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

{{<alert>}}
Note that Traefik needs to be configured in such a way, that it can see the real Client IP address, see the Helm Configuration section for a configuration that allows this.
{{</alert>}}

## Identity Management-Based Authentication

A more common approach involves user credentials, such as a username and password, ideally coupled with Multi-Factor Authentication (MFA). These credentials can be verified locally or through LDAP, SAML, or any other Identity Management System.

When combined with protocols like OAuth2 or OIDC, this can be seamlessly integrated into a service. The service can then rely on a signed token to authenticate the user appropriately.

However, this method has its drawbacks: Each application must validate the tokens or outsource this to an OAuth2 Reverse Proxy. This is feasible for browser-based applications but can be problematic for CLI-based tools, which may not support such authentication methods or are limited to specific authentication flows.

This authentication variant can be implemented individually for each service or by using the [ForwardAuth Middleware](https://doc.traefik.io/traefik/middlewares/http/forwardauth/) in combination with tools like [OAuth2-Proxy](https://oauth2-proxy.github.io/oauth2-proxy/docs/features/endpoints). Please refer to the [endpoint documentation](https://oauth2-proxy.github.io/oauth2-proxy/docs/features/endpoints) and in particular the `/oauth2/auth` endpoint.

I leave implementations details to the reader.

## TLS based

Another way to protect your sites is Mutual TLS. While it is lesser known, it is one of the most secure variant.

Mutual TLS means that not only the Browser/Client can trust the Server, but also the other way around.
Every Browser has a lot of Certification Authorities (CAs) in its truststore, which it trusts. 

During the TLS handshake the server serves a Certificate that must be signed by one of these CAs in the Browsers store. When the certificate is still valid (timestamp, not on revocation list, etc.), the Browsers accepts the Certificate an tries to establish a secure connection using a TLS handshake. Otherwise a warnig is displayed with some informations.

During this handshake, the Server can also request a Certificate from the Client that must be signed by specific CAs. This allows the Server to only establish connections to authorized clients. The Server CA and Client CA does not need to be necessarily the same, it is no problem to roll your own CA for client authentication, while using e.g. Let's Encrypt for Server certificates.

We use exactly this fact.

### Roll your own CA
There are a lot of ways to create your own CA. One easy way of creating CAs and handling certificates is the use of [XCA](https://hohnstaedt.de/xca/). It is a standalone application with an integrated database.

You can easily create a Certificate Authority, create Certificates that you sign with it and export them in a format required by the application.

For our use case, we need a CA for Client Certificates.

{{<alert>}}
While in facilities with high security requirements, you should rely on addition protection mechanism like a HSM (Hardware Security Module), this is not required for all use cases. But in all situations you should secure your CA! If someone gets access, he can create and sign additional certificates without your knowledge.

The only way to detect such a breach is to check and log certificate fingerprints. But you have to actively monitor them.
{{</alert>}}

### Traefik configuration

To change TLS configurations in Traefik, Traefik uses the `TLSOptions` CRD. If you create a `TLSOption` with the name `default`, it will be applied to all `Ingress` resources. In the cluster, you can have multiple `TLSOption` manifests and can define them individually for each `Ingress` by annotation. If you don't define any, the default `TLSOption` will be used.

#### TLSOption mit Client Auth

The following manifest restrict the available ciphers as well as require Client Authentication with certs signed by `my-ca`.

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: TLSOption
metadata:
  name: default
  namespace: default
spec:
  cipherSuites:
  - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
  - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
  - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
  clientAuth:
    clientAuthType: RequireAndVerifyClientCert
    secretNames:
    - my-ca
  minVersion: VersionTLS12
  sniStrict: true
```

The CA must be stored as a Secret which can be achieved by `kubectl create secret tls my-ca --cert=path/to/tls.cert`.

#### Services without Client Authentication

If you want to host some services without Client Authentication, you can create a dedicated `TLSOption` manifest for that

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: TLSOption
metadata:
  name: withoutcerts
  namespace: default
spec:
  cipherSuites:
  - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
  - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
  - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
  minVersion: VersionTLS12
  sniStrict: true
```

and configure it in your `Ingress` resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    traefik.ingress.kubernetes.io/router.tls.options: default-withoutcerts@kubernetescrd
  name: public-service
spec:
  rules:
  - host: domain.tld
  [...]
```

### Browser configuration

In Chrome, you can find add a `.pfx` client certificate at `chrome://settings/certificates`. You don't need to add the CA as trust anchor in the browser. If you don't use the CA for Server certificate validation, I would highly recommand not adding it there as a stolen CA could otherwise lead to a Man-in-the-Middle attacks.

Once added, you can navigate to your newly protected website and will be notified, that the server want to authenticate you, after approval you should be able to access the service.

You have now a mTLS protected service.

A similar way can be used on Android and iOS to add a Client Certificate.



## Traefik Helm Chart configuration

In addition to the general TLSOptions specified above, you can also add some additional settings to your Traefik configuration.
This will enable Traffic logging, redirect the HTTP traffic to HTTPS and binds port 80 and 443 directly on the host, as well as allowing Traefik to see the clients real IP address.

For higher security, you can bind traefik to a different port > 1024 and run it as non root or create a custom container with `setcap cap_net_bind_service=+ep /usr/local/bin/traefik` and leave `NET_BIND_SERVICE` enabled.

For k3s this can look like this:

```
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    logs:
      access:
        enabled: true
        fields:
          headers:
            defaultmode: keep
            names:
              Authorization: drop
    securityContext:
      capabilities:
        drop: [ "ALL" ]
        add: [ "NET_BIND_SERVICE" ]
      readOnlyRootFilesystem: true
      runAsGroup: 101
      runAsNonRoot: false
      runAsUser: 0
    podSecurityContext:
      seccompProfile:
        type: RuntimeDefault
      fsGroup: 101
    ports:
      traefik:
        web:
          expose: false
      web:
        redirectTo:
          port: websecure
        port: 80
        hostPort: 80
      websecure:
        port: 443
        hostPort: 443
        tls:
          enabled: true
    providers:
      kubernetesIngress:
        allowEmptyServices: true
        publishedService:
          enabled: true
    rbac:
      enabled: true
    service:
      type: ClusterIP
      spec:
        externalTrafficPolicy: null
        internalTrafficPolicy: Local
    deployment:
      kind: DaemonSet
    dashboard:
      enable: true
    hostNetwork: true
    updateStrategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 1
        maxSurge: 0
    additionalArguments:
    - "--metrics.prometheus=true"
    - "--entrypoints.websecure.proxyProtocol.trustedIPs=10.0.0.0/8,192.168.0.0/16"
    - "--entryPoints.websecure.forwardedHeaders.trustedIPs=127.0.0.1/32,10.0.0.0/8,192.168.0.0/16"
```
