---
title: "WSL: Custom Distributions"
summary: "WSL distributions from Containers, Docker in WSL, etc."
date: 2023-11-05
series: ["Windows development"]
series_order: 3
---

In this blog post, I want to show you how to build your own WSL (Windows Subsystem for Linux) distribution. If instead you prefer to use a ready-made one, I've got you covered as well. It includes:

1. Setting up a user after installation.
2. ZSH shell with the powerlevel9k theme.
3. Container tools like docker, buildah, podman, and k3s as a Kubernetes distribution.
4. Compatibility with WSLg (compatible with IntelliJ in WSL).
5. Shared /home. All my distributions share the same /home directory. This makes it easy to switch distributions, try things out, or update from one Fedora version to the next.

Ready made Distributions for _Fedora_ and _Ubuntu_ can be found at my [Github repository](https://github.com/smerschjohann/wslbox), including a concise installation guide.

{{< github repo="smerschjohann/wslbox" >}}

## Create a distribution

On the _Windows Store_ there are already a few distributions to install from. So why even bother tampering with that? 

Well, there are some reasons:

1. Those distributions don't come with Systemd preconfigured. If you want to use that, you have to enable that manually. Using Systemd comes with some benefits, it allows you to directly use daemons and services that depend on it. It is for example very easy to install Docker or even a Kubernetes Distribution (k3s), if you have working Systemd.
2. The typical WSL installation is a one time action, that can take quite some time. Especially, if all your colleaques have to do it. Why not prepare a distribution once and allow everyone to benefit from it?
3. Reproducability. With the described approach, you can just delete and reinstall a distribution many times without loosing any data (if all your data is stored in /home). This allows you to use WSL as you might already use containers.

### Background

The `wsl.exe` command allows you to import a tarball with e.g. `wsl --import Fedora c:\wsl\Fedora .\fedora.tar` as a new distribution. The mentioned command will import the _fedora.tar_ which must contain a whole root directory structure like /bin, /usr/bin etc. into a new vhdx disk which it will put into **c:\wsl\fedora**.
The new distro is then accessable by the name **Fedora** and can be started by `wsl.exe -d Fedora`. Microsoft describes this process in their [documentation](https://learn.microsoft.com/en-us/windows/wsl/use-custom-distro).

We can use this fact, to create our own distro, that comes with preinstalled tools, additional features etc.

### How to spin your own Distribution

As the `wsl.exe --import` command accepts any valid tarball with a root filesystem, it is quite easy to create such a tar with many tools. You could do it by:

1. tar'ing an actual linux system. E.g. a running VM you have somewhere around.
2. using tools like `debootstrap` which allows you to create the directory structure for Debian.
3. Using container tools like `buildah`, `docker` or `podman`.


### Build from Dockerfile

As a _Dockerfile_ allows you to use any Container image as your base, you are free to create a custom distribution using any Linux Distribution that is available as Container. That means you could use Alpine, Debian, Ubuntu, RHEL, OpenSuse and many more.

I will use Fedora for all the examples, but as stated, it is not limited to Fedora. In fact, the mentioned repository, contains also a Distribution based on Ubuntu.

Ok enough explanation, let's see a minimal Dockerfile:

```Dockerfile
FROM registry.fedoraproject.org/fedora-toolbox:38 as devbox

RUN mkdir -p /usr/lib/binfmt.d; \
    echo :WSLInterop:M::MZ::/init:PF > /usr/lib/binfmt.d/WSLInterop.conf
```

This the bare minimum that you should have inside of the custom distribution. Why that additional file? It allows the _*.exe_ interoperability. Without it, you cannot run windows executables from inside your shell. It basically allows you to execute binaries like `cmd.exe` from inside your WSL.

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
wsl.exe --import myfirstdistro c:\wsl\firstdistro .\wsl-distro.tar.gz
```

## Enhance with Systemd

WSL2 also allows distributions to use Systemd, which I highly recommend. It allows you to install `docker`, `k3s` or any other software that requires Systemd. It also allows to create background services, that are always started when you start the wsl.

To activate it, you have to create the file `/etc/wsl.conf` with the following content:
```ini
[boot]
systemd=true
```

### Home mount script
As I want to experiment with different Distributions and also allow to update my Distribution without loosing my data in /home directory, I came up with the a systemd unit file, that mounts the /home directory externally.

To do this, it uses the fact, that all WSL distributions run on the same VM. Microsoft allows to share directories in the `/mnt/wsl` directory. We use this fact in the following way:

1. Create a wsl-data distribution, that is not actively used, but holds shared data. This distribution needs at the bash and mount commands. I base this from a `busybox` container.
2. From the wsl-data distribution, bind mount a local /data directory to the /mnt/wsl/data directory.
3. On each boot of our main distribution, check if the data directory is already mounted. If not, start the wsl-data distribution and mount it accordingly.
4. Bind mount the /mnt/wsl/data/home directory to /home.

To accomplish this, the following SystemD unit file is used. It will start very early in the boot process.

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


You can integrate it in a Dockerfile by copying the files and enabling the service.

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

The repository also contains all files, that I mentioned in this post.

The build is completetly automated using *Github Actions*, so you can verify the Release builds and spin your own if you like. I will update the Distribution on major changes like new OS releases or when I think a new tool would be nice to integrate.

Feel free to create a PR for new distributions or additions to the existing once, if you have something interesting that you think everyone can participate from.

{{< github repo="smerschjohann/wslbox" >}}