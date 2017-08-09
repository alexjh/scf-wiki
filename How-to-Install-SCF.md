## Table Of Contents

* [Requirements for Kubernetes](#requirements-for-kubernetes)
* [Verifying the Kube](#verifying-the-kube)
* [Kube DNS](#kube-dns)
* [Storage Classes](#storage-classes)
* [SUSE Web UI](#suse-web-ui)
* [Helm installation](#helm-installation)
* [Installation and deployment via helm, including the generation of certs](#installation-and-deployment-via-helm-including-the-generation-of-certs)
* [Removal and Cleanup via helm](#removal-and-cleanup-via-helm)
* [CF documentation](#cf-documentation)

## Requirements for Kubernetes

The various machines (`api`, `kube`, and `node`) of the kubernetes cluster must be configured in a particular way to support the execution of `SCF`. These requirements are, in general:

* Kubernetes API versions 1.5.x-1.6.x
* Kernel parameters `swapaccount=1`
* `docker info` must not show `aufs` as the storage driver.
* `kube-dns` must be be running and `4/4 ready`. See section [Kube DNS](#kube-dns).
* Either `ntp` or `systemd-timesyncd` must be installed and active.
* The kubernetes cluster must have a storage class SCF can refer to.
  See section [Storage Classes](#storage-classes).
* Privileged container must be enabled in `kube-apiserver`. See https://kubernetes.io/docs/admin/kube-apiserver
* Privileged must be enabled in `kubelet`.
* The `TasksMax` property of the `containerd` service definition must be set to infinity.
* Helm's Tiller has to be installed and active.

|Category|Explanation|
|---|---|
|api| Requirements on the hosts for the kube master nodes (running `apiserver`) |
|kube| Requirements of the cluster itself, via `kubectl` |
|node| Requirements on the hosts for the kube worker nodes (running `kubelet`) |

An easy way of setting up a small single-machine kubernetes cluster with all the necessary properties is to use the Vagrant definition in the SCF repository. The details of this approach are explained in https://github.com/SUSE/scf/blob/develop/README.md#deploying-scf-on-vagrant

## Verifying the Kube

For ease of verification of these requirements a script ([kube-ready-state-check.sh](https://github.com/SUSE/scf/blob/develop/bin/dev/kube-ready-state-check.sh)) is made available which contains the necessary checks.

To get help invoke this script via
```
kube-ready-state-check.sh -h
```
This will especially note the various machine categories. When invoked with the name of machine category (`api`, `kube`, and `node`), i.e. like
```
kube-ready-state-check.sh kube
```
the script will run the tests applicable to the named category.
Positive results are prefixed with `Verified: `,
whereas failed requirements are prefixed with `Configuration problem detected:`.

## Kube DNS

As mentioned before, the cluster has to have an active `kube-dns`.
For CaaSP install and activate it with
```
kubectl apply \
  -f https://raw.githubusercontent.com/SUSE/caasp-services/b0cf20ca424c41fa8eaef6d84bc5b5147e6f8b70/contrib/addons/kubedns/dns.yaml
```

## Storage Classes

As mentioned before, the kubernetes cluster must have a storage class SCF can refer to so that its database components have a place for their persistent data.

This class may have any name, with the vagrant setup using `persistent`.
Important information on storage classes and how to create and configure them can be found at

* [Dynamic Provisioning and Storage Classes in Kubernetes](http://blog.kubernetes.io/2017/03/dynamic-provisioning-and-storage-classes-kubernetes.html)
* [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

Note, while the distribution comes with an example storage-class `persistent` of type `hostpath`, for use with the vagrant box, this is a toy option and should not be used with anything but the vagrant box. It is actually quite likely that whatever kube setup is used will not even support the type `hostpath` for storage classes, automatically preventing its use.

To enable hostpath support for testing, the `kube-controller-manager` can be run with the `--enable-hostpath-provisioner` command line option.

## SUSE Web UI

See https://github.com/SUSE/stratos-ui/releases
for distributions of the console. They are based on Helm charts.

## Helm installation

SCF uses Helm charts to deploy on kubernetes clusters.
To install Helm see

* https://docs.helm.sh/using_helm/#quickstart

## Installation and deployment via helm, including the generation of certs

Consider the following an annotated session demonstrating how to deploy SCF/UAA on kubernetes. In other words, explanations interspersed with example commands and vice versa.

To install SCF
* Get the distribution archive from https://github.com/SUSE/scf/releases. Create a directory and extract the archive into it.
  ```
  mkdir deploy
  cd    deploy
  unzip ../scf-X.Y.Z.linux-amd64.zip # for example
  ```
  The following relative path references will refer to files within this directory

* Unpack this archive in a directory your choice.

  We now have the helm charts for SCF and UAA in a subdirectory `helm`.
  Additional configuration files are found under `kube`.
  The `scripts` directory contains helpers for cert generation.

* __Choose__ the name of the kube storage class to use, and create the class.
   See section [Storage Classes](#storage-classes) for important notes.

   Note, while the class created here comes with the distribution it should only be used
   with the vagrant setup. It is of type `hostpath`, a toy option which is usually not
   supported by a kube setup.

   We make use of it only because the example done here is based on the vagrant setup. Note that
   the storageclass `apiVersion` used in the manifest should either be `storage.k8s.io/v1beta1` (for
   kubernetes 1.5.x) or `storage.k8s.io/v1` (for kubernetes 1.6.x)

   Use kubectl to check your kubernetes server version:
   ```
   kubectl version --short | grep "Server Version"
   ```

   For kubernetes 1.5.x:
   ```
   cat > storage-class-host-path.yaml <<END
   ---
   kind: StorageClass
   apiVersion: storage.k8s.io/v1beta1
   metadata:
     name: hostpath
   provisioner: kubernetes.io/host-path
   parameters:
     path: /tmp
   END
   ```

   For kubernetes 1.6.x:
   ```
   cat > storage-class-host-path.yaml <<END
   ---
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: hostpath
   provisioner: kubernetes.io/host-path
   parameters:
     path: /tmp
   END
   ```

   Create the class:
   ```
   kubectl create -f storage-class-host-path.yaml
   ```

* Create custom certs for the deployment by invoking the certification generator:
   ```
   mkdir certs
   ./cert-generator.sh -d cf-dev.io -n scf -o certs
   ```
  Note: Choosing a different output directory (`certs`) here will require matching changes to the commands deploying the helm charts, below.

* We now have the certificates required by the various components to talk to each other (SCF internals, UAA internals, SCF to UAA).

* Generate a values.yaml file with the required settings:
   ```
   cat > scf-config-values.yaml <<END
   env:
     # Password for the cluster
     CLUSTER_ADMIN_PASSWORD: changeme

     # Domain for SCF. DNS for *.DOMAIN must point to the kube node's
     # external ip. This must match the value passed to the
     # cert-generator.sh script.
     DOMAIN: cf-dev.io

     # Password for SCF to authenticate with UAA
     UAA_ADMIN_CLIENT_SECRET: uaa-admin-client-secret

     # UAA host/port that SCF will talk to. The example values here are
     # for the UAA deployment included with the SCF distribution.
     UAA_HOST: uaa.cf-dev.io
     UAA_PORT: 2793
   kube:
     # The IP address assigned to the kube node. The example value here
     # is what the vagrant setup assigns
     external_ip: 192.168.77.77
   END
   ```

* Use Helm to deploy UAA. Remember that the [previous section](#helm-installation) gave a reference to the Helm documentation explaining how to install Helm itself. Remember also that in the Vagrant-based setup `helm` is already installed and ready.
   ```
   helm install helm/uaa \
     --set kube.storage_class.persistent=hostpath \
     --namespace uaa \
     --values certs/uaa-cert-values.yaml \
     --values scf-config-values.yaml
   ```

* With UAA deployed, use Helm to deploy SCF.
   ```
   helm install helm/cf \
     --set kube.storage_class.persistent=hostpath \
     --namespace scf \
     --values certs/scf-cert-values.yaml \
     --values scf-config-values.yaml
   ```

* Wait for everything to be ready:
   ```
   watch -c 'kubectl get pods --all-namespaces'
   ```
  Stop watching when all pods show state `Running` and Ready is `n/n` (instead of `k/n`, `k < n`).

* Basic operation of the deployed SCF can be verified by running the CF smoke tests.
  To invoke the tests run the command
   ```
   kubectl create \
      --namespace=scf \
      --filename="kube/cf/bosh-task/smoke-tests.yml"

   # Wait for completion
   kubectl logs --follow --namespace=scf smoke-tests
   ```

* If the deployed SCF is not intended as a production system then its operation can be
  verified further by running the CF acceptance tests.

  > CAUTION: tests are only meant for acceptance environments, and while they attempt to clean up after themselves, no guarantees are made that they won't change the state of the system in an undesirable way.
  > -- https://github.com/cloudfoundry/cf-acceptance-tests/

  To invoke the tests run the command
   ```
   kubectl create \
      --namespace=scf \
      --filename="kube/cf/bosh-task/acceptance-tests.yml"

   # Wait for completion
   kubectl logs --follow --namespace=scf acceptance-tests
   ```

* Pulling everything together we have
   ```
   wget URL-TO-SCF-ZIP-DISTRIBUTION

   mkdir deploy
   cd    deploy
   unzip ../scf-X.Y.Z.linux-amd64.zip # example

   cat > storage-class-host-path.yaml <<END
   ---
   kind: StorageClass
   # NOTE: For kubernetes 1.5, this should be set to
   #       apiVersion: storage.k8s.io/v1beta1
   apiVersion: storage.k8s.io/v1
   metadata:
     name: hostpath
   provisioner: kubernetes.io/host-path
   parameters:
     path: /tmp
   END
   kubectl create -f storage-class-host-path.yaml

   mkdir certs
   ./cert-generator.sh -d cf-dev.io -n scf -o certs

   cat > scf-config-values.yaml <<END
   env:
     # Password for the cluster
     CLUSTER_ADMIN_PASSWORD: changeme

     # Domain for SCF. DNS for *.DOMAIN must point to the kube node's
     # external ip. This must match the value passed to the
     # cert-generator.sh script.
     DOMAIN: cf-dev.io

     # Password for SCF to authenticate with UAA
     UAA_ADMIN_CLIENT_SECRET: uaa-admin-client-secret

     # UAA host/port that SCF will talk to. The example values here are
     # for the UAA deployment included with the SCF distribution.
     UAA_HOST: uaa.cf-dev.io
     UAA_PORT: 2793
   kube:
     # The IP address assigned to the kube node. The example value here
     # is what the vagrant setup assigns
     external_ip: 192.168.77.77
   END

   helm install helm/uaa \
     --set kube.storage_class.persistent=hostpath \
     --namespace uaa \
     --values certs/uaa-cert-values.yaml \
     --values scf-config-values.yaml

   helm install helm/cf \
     --set kube.storage_class.persistent=hostpath \
     --namespace scf \
     --values certs/scf-cert-values.yaml \
     --values scf-config-values.yaml

    # Wait for readiness
    watch -c 'kubectl get pods --all-namespaces'

    kubectl create \
      --namespace=scf \
      --filename="kube/cf/bosh-task/smoke-tests.yml"

    # Wait for completion
    kubectl logs --follow --namespace=scf smoke-tests

    kubectl create \
      --namespace=scf \
      --filename="kube/cf/bosh-task/acceptance-tests.yml"

    # Wait for completion
    kubectl logs --follow --namespace=scf acceptance-tests
   ```

* There are some slight changes when running SCF on CaaSP. Main difference in the configuration are domain, ip address, and storageclass. Related to that, there are additional commands to generate and feed CEPH secrets into the kube, for use by the storageclass:
   ```
   cat > scf-config-values.yaml <<END
   env:
     # Password for the cluster
     CLUSTER_ADMIN_PASSWORD: changeme

     # Domain for SCF. DNS for *.DOMAIN must point to the kube node's
     # external ip. This must match the value passed to the
     # cert-generator.sh script.
     DOMAIN: 10.0.0.154.nip.io

     # Password for SCF to authenticate with UAA
     UAA_ADMIN_CLIENT_SECRET: uaa-admin-client-secret

     # UAA host/port that SCF will talk to. The example values here are
     # for the UAA deployment included with the SCF distribution.
     UAA_HOST: uaa.10.0.0.154.nip.io
     UAA_PORT: 2793
   kube:
     # The IP address assigned to the kube node. The example value here
     # is what the vagrant setup assigns
     external_ip: 10.0.0.154
   END

   mkdir certs
   ./cert-generator.sh -d 10.0.0.154.nip.io -n scf -o certs

   kubectl create namespace uaa
   kubectl get secret ceph-secret -o json --namespace default | jq ".metadata.namespace = \"uaa\"" | kubectl create -f -

   helm install helm/uaa \
     --set kube.storage_class.persistent=fast \
     --namespace uaa \
     --values certs/uaa-cert-values.yaml \
     --values scf-config-values.yaml

   kubectl create namespace scf
   kubectl get secret ceph-secret -o json --namespace default | jq ".metadata.namespace = \"scf\"" | kubectl create -f -

   helm install helm/cf \
     --set kube.storage_class.persistent=fast \
     --namespace scf \
     --values certs/scf-cert-values.yaml \
     --values scf-config-values.yaml
   ```

## Removal and Cleanup via helm

First delete the running system at the kube level

```
    kubectl delete namespace uaa
    kubectl delete namespace scf
```
This will especially remove all the associated volumes as well.

After that use `helm list` to locate the releases for the SCF and UAA charts and `helm delete` to remove them at helm's level as well.

## CF documentation

* https://docs.cloudfoundry.org/
