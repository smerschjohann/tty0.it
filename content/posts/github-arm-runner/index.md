---
title: "Custom Github Runner"
summary: "Setup hardened custom Github Runner on Kubernetes"
date: 2023-11-01
weight: 1
chapter: false
---

GitHub offers managed runners for GitHub Actions/Workflows that are sufficient for most use cases, but not all. There are scenarios where these managed runners reveal their limitations:

1. Building large applications requiring more than 7 GB of RAM.
2. Building for different architectures.
3. Accessing local resources within a private VPC.

For the first scenario, GitHub's larger runners, which provide up to 256 GB of RAM, are available for a fee.

I encountered the second scenario. While cross-compiling with `qemu-user-static` is possible, it significantly increases build times—a five-minute build could extend to nearly an hour.

For macOS, an ARM-based machine is now available in [Beta](https://docs.github.com/en/actions/using-github-hosted-runners/about-larger-runners/about-larger-runners#about-macos-larger-runners), but for Linux there is no solution.

Since GitHub does not currently support ARM-based systems for Linux, creating your own runner is necessary. GitHub offers an executable for self-hosting runners that you can deploy on any server.

## Kubernetes

There is a dedicated Kubernetes Operator for dynamically spinning up pods and registering these new Github Runners on demand.

For simpler requirements, this may be overkill. Conversely, running the executable directly on a VM is not advisable, as build jobs executed in the runner's user context could compromise the VM's integrity. Builds could also leak state to later builds.

A more secure approach is to run the GitHub Runner within Kubernetes with additional restrictions.

## Container Support
By default, Actions cannot use Docker unless the pod is given privileged access and Docker is run as a sidecar.

This setup is acceptable if you trust all the code and dependencies executed on the runner, such as npm packages.

However, a safer method is:

1. Prohibit `Dockerfile` container builds using Docker entirely.
2. Allow only `buildah` for building container images.
3. Permit `podman` for running test containers.

`buildah` and `podman` have different system requirements. Configuring buildah with the _VFS driver_ and _chroot isolation mode_ is relatively straightforward.

To improve build performance - since VFS duplicates layers at each step unless layering is disabled - it is possible to configure the system without granting the container privileged access.

## Github Runner with Podman Support

Below is a functional Kubernetes deployment manifest enabling `podman` with native kernel `overlay` storage. 


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-runner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: github-runner
  template:
    metadata:
      annotations:
        container.apparmor.security.beta.kubernetes.io/github-runner: unconfined
      labels:
        app: github-runner
    spec:
      automountServiceAccountToken: false
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - my-arm-host
      containers:
      - command:
        - /usr/bin/sh
        - -c
        - cd /home/coder/actions-runner; ./run.sh
        env:
        - name: _BUILDAH_STARTED_IN_USERNS
        - name: BUILDAH_ISOLATION
          value: chroot
        image: ghcr.io/smerschjohann/containers/fedbox:latest
        imagePullPolicy: Always
        name: github-runner
        resources:
          limits:
            cpu: "3"
            memory: 10Gi
          requests:
            cpu: 100m
            memory: 1Gi
        securityContext:
          seccompProfile:
            type: Unconfined
        volumeMounts:
        - mountPath: /home/coder
          name: containers
        - mountPath: /dev/net/tun
          name: dev-tun
      enableServiceLinks: false
      hostname: selfhosted-arm64
      initContainers:
      - command:
        - /bin/sh
        - -c
        - |
          mkdir -p /home/coder/.config/containers; echo -e "[containers]\nvolumes = [\n\t\"/proc:/proc\",\n]\ndefault_sysctls = []" > /home/coder/.config/containers/containers.conf; cp -av /runner/* /home/coder; chown -R 1000:1000 /home/coder
        image: busybox:1.28
        imagePullPolicy: IfNotPresent
        name: init-github
        resources: {}
        securityContext:
          runAsUser: 0
        volumeMounts:
        - mountPath: /runner
          name: runner
        - mountPath: /home/coder
          name: containers
      volumes:
      - hostPath:
          path: /opt/github-runner
          type: ""
        name: runner
      - emptyDir:
          sizeLimit: 30Gi
        name: containers
      - hostPath:
          path: /dev/net/tun
          type: CharDevice
        name: dev-tun
```

### Disabling Security Profiles

It is necessery to disable Seccomp, SELinux and AppArmor for these build containers to prevent them from blocking storage access or any other Kernel feature required for `buildah` and `podman`. Crafting a custom SELinux profile that accommodates `podman` requirements is complex, but would be best.

From a security perspective, disabling these measures is not ideal. Future developments may allow for more restrictive profiles than _Unconfined_.

It's noteworthy that until Kubernetes 1.19, Unconfined was the default mode for all containers.
Kubernetes 1.27 introduced the option to set RuntimeDefault as the default for all Pods.

One good thing: Even with these settings, the pod does not run `elevated`, there are still a lot of restrictions in place.

#### Relevant part in manifest

```yaml
    annotations:
        container.apparmor.security.beta.kubernetes.io/github-runner: unconfined
```

```yaml
    securityContext:
        seccompProfile:
            type: Unconfined
```

### Volume mount
For the `overlay` storage driver of `podman` and `buildah`, the underlying filesystem must be of a different type than `overlay`, otherwise the Linux-Kernel rejects the mount.

This would be the case, if we do not mount an additional volume. But by doing so, we can make the `overlay` driver happy:

#### Relevant part in manifest

```yaml
        volumeMounts:
        - mountPath: /home/coder
          name: containers
          [...]
      volumes:
      - emptyDir:
          sizeLimit: 30Gi
        name: containers
```

### Networking

Certain kernels prevent creating network namespaces inside containers when user namespaces are used. To enable networking, it's necessary to pass the host's /proc to the child container, as configured in the init container.

`podman` also requires the `/dev/net/tun` device for slirp4netns to work.

#### Relevant part in manifest

```ini
[containers]
volumes = [
        "/proc:/proc",
]
default_sysctls = []
```

```yaml
      - hostPath:
          path: /dev/net/tun
          type: CharDevice
        name: dev-tun
```


## Network Policies

With the Deployment mentioned above, it is possible to run a Github Runner with less privileges. But the Container would still have complete access to the local network.

As it is in general best practice, we can set _NetworkPolicies_ to further reduce the risk of the deployment.


### Default Deny Policy
Start with a default deny policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```


### Allow DNS Policy

As DNS is required, allow it specifically.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns
spec:
  egress:
  - ports:
    - port: 53
      protocol: UDP
  podSelector: {}
  policyTypes:
  - Egress
```

### Allow Internet-only Policy

Block local network traffic and only allow Internet traffic. The Policy below will restrict any IANA networks including Carrier-Grade NAT and the link local network:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-internet
spec:
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
        - 100.64.0.0/10
        - 169.254.0.0/16
  podSelector: {}
  policyTypes:
  - Egress
```

{{< alert >}}
Malicious code could still do harm, this is not a complete protection against all possible threats!
{{< /alert >}}


## Example

Inside of your Github Action you can use `podman` like this:

```bash
$ podman run -i --net=host \
  ghcr.io/smerschjohann/containers/fedbox:latest \
  /bin/bash -c "echo hello world"
hello world
```

Using slirp4netns will give you a warning, but will still work:

```bash
$ podman run -i \
  ghcr.io/smerschjohann/containers/fedbox:latest \
  /bin/bash -c "echo hello world"
WARN[0000] failed to set net.ipv6.conf.default.accept_dad sysctl: open /proc/sys/net/ipv6/conf/default/accept_dad: read-only file system 
hello world
```
