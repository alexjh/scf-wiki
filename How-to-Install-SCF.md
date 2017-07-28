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
kube-ready-state-check.sh -v
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
Import information on storage classes and how to create and configure them can be found at

* [Dynamic Provisioning and Storage Classes in Kubernetes](http://blog.kubernetes.io/2017/03/dynamic-provisioning-and-storage-classes-kubernetes.html)
* [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

Note, while the distribution comes with an example storage-class `persistent` of type `hostpath`, for use with the vagrant box, this is a toy option and should not be used with anything but the vagrant box. It is actually quite likely that whatever kube setup is used will not even support the type `hostpath` for storage classes, automatically preventing its use.

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
* __Choose__ the `DOMAIN` of SCF and use standard tools to set up the DNS
  so the IP address where SCF is exposed is reachable under `*.DOMAIN`.
   ```
   export DOMAIN=cf-dev.io
   ```
* __Choose__ the `NAMESPACE` of SCF, i.e. the kube namespace the SCF components will run in later.
   ```
   export NAMESPACE=scf
   ```
* __Choose__ the `NAMESPACE` of UAA, i.e. the kube namespace the UAA components will run in later.
  This assumes that we use the UAA deployment coming with the distribution.
   ```
   export UAA_NAMESPACE=uaa
   ```
* __Choose__ a password for your SCF deployment.
   ```
   export CLUSTER_ADMIN_PASSWORD=changeme
   ```
* __Choose__ the UAA the SCF should talk to. In this example we are using the UAA deployment coming with the SCF distribution.
   ```
   export UAA_HOST=uaa.${DOMAIN}
   export UAA_PORT=2793
   export UAA_ADMIN_CLIENT_SECRET=uaa-admin-client-secret
   ```
   These variables hold the configuration SCF has to know about the UAA to talk to, i.e. location (host, port), and authentication

* __Choose__ the IP address of the kube host.
   ```
   export KUBE_HOST_IP=192.168.77.77
   ```
   The IP address chosen here is what the vagrant setup uses.

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
   kubernetes 1.5.x) or `v1` (for kubernetes 1.6.x)
   ```
   STORAGECLASS=${STORAGECLASS:-hostpath}   
   KUBE_MINOR_VERSION=$(kubectl version --short | grep -oE "Server Version: v1\.[0-9]+" |    sed 's/Server Version: v1\.//g')
   if [[ $KUBE_MINOR_VERSION -eq 5 ]]; then
       APIVERSION=storage.k8s.io/v1beta1
   elif [[ $KUBE_MINOR_VERSION -eq 6 ]]; then
       APIVERSION=storage.k8s.io/v1
   fi
   kubectl get storageclass ${STORAGECLASS} 2>/dev/null || {
       sed kube/uaa/kube-test/storage-class-host-path.yml \
           -e "s/^  name:.*/  name: ${STORAGECLASS}/" \
           -e "s@^apiVersion:.*@apiVersion: ${APIVERSION}@g" | \
       kubectl create -f -
   }
   ```

* Save all choices to environment variables.
  These are used in the coming commands.

* Create custom certs for the deployment by invoking the certification generator:
   ```
   mkdir certs
   ./cert-generator.sh -d ${DOMAIN} -n ${NAMESPACE} -o certs
   ```
  Note: Choosing a different output directory (`certs`) here will require matching changes to the commands deploying the helm charts, below.

* We now have the certificates required by the various components to talk to each other (SCF internals, UAA internals, SCF to UAA).

* Use Helm to deploy UAA. Remember that the [previous section](#helm-installation) gave a reference to the Helm documentation explaining how to install Helm itself. Remember also that in the Vagrant-based setup `helm` is already installed and ready.
   ```
   helm install helm/uaa \
     --set kube.storage_class.persistent=${STORAGECLASS} \
     --namespace ${UAA_NAMESPACE} \
     --values certs/uaa-cert-values.yaml \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}" \
     --set "kube.external_ip=${KUBE_HOST_IP}"
   ```

* With UAA deployed, use Helm to deploy SCF.
   ```
   helm install helm/cf \
     --set kube.storage_class.persistent=${STORAGECLASS} \
     --namespace ${NAMESPACE} \
     --values certs/scf-cert-values.yaml \
     --set "env.CLUSTER_ADMIN_PASSWORD=$CLUSTER_ADMIN_PASSWORD" \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}" \
     --set "env.UAA_HOST=${UAA_HOST}" \
     --set "env.UAA_PORT=${UAA_PORT}" \
     --set "kube.external_ip=${KUBE_HOST_IP}"
   ```

* If the deployed SCF is not intended as a production system then its operation can be
  verified by running the CF smoke and acceptance tests (in this order).

  > CAUTION: tests are only meant for acceptance environments, and while they attempt to clean up after themselves, no guarantees are made that they won't change the state of the system in an undesirable way.
  > -- https://github.com/cloudfoundry/cf-acceptance-tests/

  To invoke the tests run the commands
   ```
   kubectl create \
      --namespace="${NAMESPACE}" \
      --filename="kube/cf/bosh-task/smoke-tests.yml"
   ```
   and
   ```
   kubectl create \
      --namespace="${NAMESPACE}" \
      --filename="kube/cf/bosh-task/acceptance-tests.yml"
   ```

* Pulling everything together we have
   ```
   export DOMAIN=cf-dev.io
   export NAMESPACE=scf
   export UAA_NAMESPACE=uaa
   export CLUSTER_ADMIN_PASSWORD=changeme
   export UAA_HOST=uaa.${DOMAIN}
   export UAA_PORT=2793
   export UAA_ADMIN_CLIENT_SECRET=uaa-admin-client-secret
   export KUBE_HOST_IP=192.168.77.77
   STORAGECLASS=${STORAGECLASS:-hostpath}   
   KUBE_MINOR_VERSION=$(kubectl version --short | grep -oE "Server Version: v1\.[0-9]+" |    sed 's/Server Version: v1\.//g')
   if [[ $KUBE_MINOR_VERSION -eq 5 ]]; then
       APIVERSION=storage.k8s.io/v1beta1
   elif [[ $KUBE_MINOR_VERSION -eq 6 ]]; then
       APIVERSION=storage.k8s.io/v1
   fi
   kubectl get storageclass ${STORAGECLASS} 2>/dev/null || {
       sed kube/uaa/kube-test/storage-class-host-path.yml \
           -e "s/^  name:.*/  name: ${STORAGECLASS}/" \
           -e "s@^apiVersion:.*@apiVersion: ${APIVERSION}@g" | \
       kubectl create -f -
   }
   wget URL-TO-SCF-ZIP-DISTRIBUTION

   mkdir deploy
   cd    deploy
   unzip ../scf-X.Y.Z.linux-amd64.zip # example

   mkdir certs
   ./cert-generator.sh -d ${DOMAIN} -n ${NAMESPACE} -o certs

   helm install helm/uaa \
     --set kube.storage_class.persistent=${STORAGECLASS} \
     --namespace ${UAA_NAMESPACE} \
     --values certs/uaa-cert-values.yaml \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}" \
     --set "kube.external_ip=${KUBE_HOST_IP}"

   helm install helm/cf \
     --set kube.storage_class.persistent=${STORAGECLASS} \
     --namespace ${NAMESPACE} \
     --values certs/scf-cert-values.yaml \
     --set "env.CLUSTER_ADMIN_PASSWORD=$CLUSTER_ADMIN_PASSWORD" \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}" \
     --set "env.UAA_HOST=${UAA_HOST}" \
     --set "env.UAA_PORT=${UAA_PORT}" \
     --set "kube.external_ip=${KUBE_HOST_IP}"

    # Wait for completion

    kubectl create \
      --namespace="${NAMESPACE}" \
      --filename="kube/cf/bosh-task/smoke-tests.yml"

    # Wait for completion

    kubectl create \
      --namespace="${NAMESPACE}" \
      --filename="kube/cf/bosh-task/acceptance-tests.yml"
   ```

* Vlad used an alternate script for SCF on CaaSP. Main difference in the configuration are domain, ip address and storageclass. Related to that his code also contains additional commands to generate and feed CEPH secrets into the kube, for use by the storageclass
   ```
   export DOMAIN=10.84.75.154.nip.io
   export NAMESPACE=scf
   export UAA_NAMESPACE=uaa
   export CLUSTER_ADMIN_PASSWORD=changeme
   export UAA_HOST=uaa.${DOMAIN}
   export UAA_PORT=2793
   export UAA_ADMIN_CLIENT_SECRET=uaa-admin-client-secret
   export KUBE_HOST_IP=10.84.75.154
   export STORAGECLASS=fast

   mkdir certs
   ./cert-generator.sh -d ${DOMAIN} -n ${NAMESPACE} -o certs

   kubectl create namespace "$UAA_NAMESPACE"
   kubectl get secret ceph-secret -o json --namespace default | jq ".metadata.namespace = \"${UAA_NAMESPACE}\"" | kubectl create -f -

   helm install helm/uaa \
     --set kube.storage_class.persistent=${STORAGECLASS} \
     --namespace ${UAA_NAMESPACE} \
     --values certs/uaa-cert-values.yaml \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}" \
     --set "kube.external_ip=${KUBE_HOST_IP}"

   kubectl create namespace "$NAMESPACE"
   kubectl get secret ceph-secret -o json --namespace default | jq ".metadata.namespace = \"${NAMESPACE}\"" | kubectl create -f -

   helm install helm/cf \
     --set kube.storage_class.persistent=${STORAGECLASS} \
     --namespace ${NAMESPACE} \
     --values certs/scf-cert-values.yaml \
     --set "env.CLUSTER_ADMIN_PASSWORD=$CLUSTER_ADMIN_PASSWORD" \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}" \
     --set "env.UAA_HOST=${UAA_HOST}" \
     --set "env.UAA_PORT=${UAA_PORT}" \
     --set "kube.external_ip=${KUBE_HOST_IP}"
   ```

## Removal and Cleanup via helm

First delete the running system at the kube level

```
    kubectl delete namespace $UAA_NAMESPACE
    kubectl delete namespace $NAMESPACE
```
This will especially remove all the associated volumes as well.

After that use `helm list` to locate the releases for the SCF and UAA charts and `helm delete` to remove them at helm's level as well.

## CF documentation

* https://docs.cloudfoundry.org/
