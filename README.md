# buildah-rootless-containers-openshift
build rootless containers on OpenShift 4.x

#### Make builder image

Containerfile

~~~
# Reference: https://github.com/redhat-actions/openshift-actions-runners/blob/main/buildah/Containerfile
ARG BASE_IMG=registry.access.redhat.com/ubi8/ubi
FROM $BASE_IMG AS buildah-runner

RUN useradd buildah; echo buildah:10000:5000 > /etc/subuid; echo buildah:10000:5000 > /etc/subgid;

# https://github.com/containers/buildah/blob/main/docs/tutorials/05-openshift-rootless-build.md
# https://github.com/containers/buildah/blob/master/contrib/buildahimage/stable/Dockerfile
# https://github.com/containers/buildah/issues/1011
# https://github.com/containers/buildah/issues/3053

RUN dnf -y update && \
    dnf -y install xz slirp4netns buildah podman fuse-overlayfs shadow-utils --exclude container-selinux && \
    dnf -y reinstall shadow-utils && \
    dnf clean all

RUN chgrp -R 0 /etc/containers/ && \
    chmod -R a+r /etc/containers/ && \
    chmod -R g+w /etc/containers/

ENV BUILDAH_ISOLATION=chroot
ENV BUILDAH_LAYERS=true

ADD https://raw.githubusercontent.com/containers/buildah/master/contrib/buildahimage/stable/containers.conf /etc/containers/

RUN chgrp -R 0 /etc/containers/ && \
    chmod -R a+r /etc/containers/ && \
    chmod -R g+w /etc/containers/

# Use VFS since fuse does not work
# https://github.com/containers/buildah/blob/master/vendor/github.com/containers/storage/storage.conf
RUN mkdir -vp /home/buildah/.config/containers && \
    printf '[storage]\ndriver = "vfs"\n' > /home/buildah/.config/containers/storage.conf && \
    chown -Rv buildah /home/buildah/.config/

USER buildah
WORKDIR /home/buildah
~~~

#### Create a service account which is solely used for image building.

~~~
oc create -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: buildah-sa
EOF
~~~

You need to assign it the ability to run as the standard `anyuid` [SCC](https://docs.openshift.com/container-platform/4.3/authentication/managing-security-context-constraints.html).

~~~
oc adm policy add-scc-to-user anyuid -z buildah-sa
~~~

This will give the container `cap_kill`, `cap_setgid`, and `cap_setuid` capabilities which are extras compared to the restricted SCC. Note that `cap_kill` is dropped by the DeploymentConfig, but the two others are required to execute commands with different user ids as an image is built.

With this in place, when you get the Pod running, its YAML state will contain:

#### Create DeploymentConfig

This is a simple DC just to get the container running.

Note that it drops CAP_KILL which is not required.
~~~
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: buildah
spec:
  selector:
    app: image-builder
  replicas: 1
  template:
    metadata:
      labels:
        app: image-builder
    spec:
      serviceAccount: buildah-sa
      containers:
      - name: buildah
        image: quay.io/canit0/buildah
        command: ['sh', '-c', 'sleep 36000']
        securityContext:
          capabilities:
            drop:
              - KILL
~~~
