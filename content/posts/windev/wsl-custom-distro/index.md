---
title: "WSL: Custom Distributions"
summary: "WSL distributions from Containers, Docker in WSL, etc."
date: 2023-10-11
draft: false
series: ["Windows development"]
series_order: 3
---

This blog post is for all of us still dependend on Windows for our day to day job.

In this post, I want to show you, how you can create a WSL distribution, allow to share the home directory between multiple distributions and enable moby/docker in the WSL.

As I like to use Fedora, I present all examples using Fedora, but Debian, Ubuntu or any other major distribution should work as well.


## Create a distribution

On the Windows Store there are already a few distributions ready to go. But what if they don't fit what you want or you want to provide your colleaques with a variant with preinstalled packages?

You can create your own Distribution!

The `wsl.exe` command allows you to import a tarball with e.g. `wsl --import Fedora c:\wsl\Fedora .\fedora.tar` as a new distribution. The mentioned command will import the _fedora.tar_ which must contain a whole root directory structure like /bin, /usr/bin etc. into a new vhdx drive which it will put into **c:\wsl\fedora**. The new distro is then accessable by the name **Fedora**. Microsoft describes this process on a [dedicated page](https://learn.microsoft.com/en-us/windows/wsl/use-custom-distro).

We can use this fact, to create our own distro, that comes with preinstalled tools, additional features etc. I did exactly that with a custom distro with:

1. Docker / K3s preinstalled
2. ZSH with customization already applied
3. Development environment like IntelliJ (using WSLg) and Code.
4. Custom Systemd Unit to allow sharing the /home directory between WSL distributions.

So if you don't want to create your own distro, but use that one, feel free to use it. An installation description can be found in the _README.md_.

{{< github repo="smerschjohann/wslbox" >}}

### How to create?

As the `wsl.exe --import` command accepts any valid tarball with a root filesystem, it is quite easy to create such a tar with many tools. You could do it by:

1. tar'ing an actual linux system. E.g. a running VM you have somewhere around.
2. using tools like `debootstrap` which allow to create the directory structure for Debian.
3. Using container tools like `docker` or `podman`.

We are using `podman` for this use case as it allows in combination with `skopeo` to create a tarball from any _Dockerfile_.


### Build from Dockerfile

As a _Dockerfile_ allows you to use any Container image as your base, you are free to create a custom distribution using any Linux Distribution that is available as Container. That means you could use Alpine, Debian, Ubuntu, RHEL, OpenSuse and many more.

I will use Fedora for all the examples, but as stated, it is not limited to Fedora. In the mentioned repository of mine, I even maintain another Distribution based on Debian.

Ok enough explanation, let's see a minimal Dockerfile:

```Dockerfile
FROM registry.fedoraproject.org/fedora-toolbox:38 as devbox

RUN mkdir -p /usr/lib/binfmt.d; \
    echo :WSLInterop:M::MZ::/init:PF > /usr/lib/binfmt.d/WSLInterop.conf
```

This the bare minimum that you should have inside of the custom distribution. Why that additional file? It allows the _*.exe_ interoperability. Without it, you cannot run windows executables from inside your shell. It basically allows you to execute executables like `cmd.exe` from inside your WSL.

To build the Dockerfile into a tarball that can be imported by `wsl.exe`, you can use `podman`, `buildah` or `docker`. I recommend `buildah` in pipelines, as it allows building it with least privileges. Please see my upcoming post of how to allow `buildah` builds on Openshift and Kubernetes using the `overlay` storage driver.

#### podman

```bash
podman build . -t wtest:latest
podman unshare
IMG_MOUNT=$(podman image mount localhost/wtest:latest)
tar czavf wsl-distro.tar.gz -C $IMG_MOUNT .
podman image unmount localhost/wtest:latest
exit
```

#### buildah

```bash
buildah bud -t wtest:latest
buildah from --name=wslexport localhost/wtest:latest
buildah unshare --mount wslexport \
          sh -c 'tar czavf wsl-distro.tar.gz -C $wslexport .'
buildah rm wslexport
buildah rmi localhost/wtest:latest
```

#### docker

```bash
docker build . -t devbox:latest
docker rm devdevdev || true
docker create --name devdevdev devbox:latest
docker export devdevdev -o devbox-wsl.tar
docker rm devdevdev
docker rmi devbox:latest
```

The resulting wsl-distro.tgz can now be imported as WSL distribution:

```
gunzip wsl-distro.tar.gz
wsl.exe --import myfirstdistro c:\wsl\firstdistro .\wsl-distro.tar 
```

## Enhance with systemd

WSL2 also allows distributions to use systemd, which I highly recommend. It allows you to install docker, k3s or any other software that requires systemd. It also allows to create background services, that are always started when you start the wsl.

To activate it, you have to create the file `/etc/wsl.conf` with the following content:
```ini
[boot]
systemd=true
```

### Home mount script
As I want to experiment with different distributions and also allow to update my distribution without loosing my /home directory, I came up with the a systemd unit file, that mounts the /home directory externally.

To do this, it uses the fact, that all WSL distributions run on the same VM. Microsoft allows to share directories in the `/mnt/wsl` directory. We use this fact in the following way:

1. Create a wsl-data distribution, that is not actively used, but holds shared data. This distribution needs at the bash and mount commands. I base this from a `busybox` container.
2. From the wsl-data distribution, bind mount a local /data directory to the /mnt/wsl/data directory.
3. On each boot of our main distribution(s), check if the data is already mounted. If not, start the wsl-data distribution and mount it accordingly.
4. Bind mount the /mnt/wsl/data/home directory to /home.

To accomplish this, the following SystemD unit file is used. It is configured, to start very early in the boot process.

```ini
[Unit]
Description=Mount Home Script
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/mount-home.sh

[Install]
WantedBy=multi-user.target
```

The `mount-home.sh` does most of the work. It checks if `/mnt/wsl/data` is already mounted by any WSL distribution, if not it starts the `wsl-data` distribution and waits for it to become available.

Next the shared directory is mounted to `/home`.


```bash
#!/bin/bash

set -eo pipefail

startWslData() {
    /mnt/c/Windows/system32/wsl.exe -d wsl-data << EOF
    mkdir -p /data;
    mkdir -p /mnt/wsl/data;
    umount /mnt/wsl/data 2> /dev/null || true; 
    mount -o bind /data /mnt/wsl/data;
    chmod a+rwx /mnt/wsl/data;
    echo mounted.
EOF
}

if ! grep -q "/mnt/wsl/data" /proc/mounts; then
    startWslData

    while true; do
        sleep 3
        if grep -q "/mnt/wsl/data" /proc/mounts; then
            break
        fi
    done
fi

mkdir -p /mnt/wsl/data/home
mount -o bind /mnt/wsl/data/home /home
```

I use this as a daily driver without any issues.


You can integrate it in a Dockerfile like this:

```Dockerfile
FROM registry.fedoraproject.org/fedora-toolbox:38 as devbox

# NOTE:
# the amount of layers is not important!
# We export it as a flat filesystem tarball. 
# So feel free to add as many layers as you want

RUN dnf install -y systemd

COPY --chown=0:0 system_files/common /
COPY --chown=0:0 system_files/fedora /
RUN chmod u+rwx /usr/sbin/*
RUN systemctl enable mount-home.service

# disable selinux as parts does not seem to be compatible with WSL2
RUN sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

RUN echo :WSLInterop:M::MZ::/init:PF > /usr/lib/binfmt.d/WSLInterop.conf

# final cleanup
RUN dnf clean all && rm -rf /var/cache/yum
```

The `wsl-data` distribution can be as simple as a default `busybox` image.

```bash
docker rm devdata || true
docker create --name devdata busybox:latest
docker export devdata -o wsl-data.tar
docker rm devdata
```


## My distribution

Now you know, how to create your own distribution. If you don't want to do that, but still profit from the `mount-home.service`, you can use the WSL distribution that I share in my repository.

The repository also contains all files, that I mentioned in this post. In addition to the mentioned parts, it also adds:

1. K3s (disabled by default, start with `systemctl start k3s`).
2. Docker (can be disabled by `systemctl disable docker`)
3. Necessary tools for WSLg and IntelliJ. Please raise an Issue or PR, if you want variants with preinstalled IntelliJ.
4. A lot of utility tools for Kubernetes etc.

The build is completetly automated using *Github Actions*, so you can verify the Release builds. The distribution is automatically build every two weeks, so that it includes the latest updates.

{{< github repo="smerschjohann/wslbox" >}}