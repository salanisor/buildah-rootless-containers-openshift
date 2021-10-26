# buildah-rootless-containers-openshift
build rootless containers on OpenShift 4.x

#### Make builder image

Containerfile

~~~
ARG BASE_IMG=quay.io/buildah/stable
FROM $BASE_IMG

RUN dnf -y update && \
    dnf clean all

ENV BUILDAH_ISOLATION=chroot
ENV BUILDAH_LAYERS=true
ENV STORAGE_DRIVER=vfs

USER build
WORKDIR /home/build
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
