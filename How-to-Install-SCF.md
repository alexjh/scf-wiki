## Requirements for Kube

The various machines (`api`, `kube`, and `node`) of the kubernetes cluster must be configured in a particular way to support the execution of `SCF`. These requirements are, in general:

* Kernel parameters `swapaccount=1`
* `docker info` must not show `aufs`.
* `kube-dns` must be be running and `4/4 ready`.
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

For ease of verification of these requirements a script (`kube-ready-state-check.sh`) is made available which contains the necessary checks.

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

* __Choose__ the name of the kube storage class to use, and create the class.
   See section [Storage Classes](#storage-classes) for important notes.
   ```
   export STORAGECLASS=persistent
   kubectl get storageclass persistent 2>/dev/null || {
       perl -p -e 's@storage.k8s.io/v1beta1@storage.k8s.io/v1@g' \
           "kube/uaa/kube-test/storage-class-host-path.yml" | \
       kubectl create -f -
   }
   ```
   Note, while the class created here comes with the distribution it should only be used
   with the vagrant setup. It is of type `hostpath`, a toy option which is usually not
   supported by a kube setup.

   We make use of it only because the example done here is based on the vagrant setup.

* Save all choices to environment variables.
  These are used in the coming commands.

* Get the distribution archive from https://github.com/SUSE/scf/releases
  For example
   ```
   wget https://github.com/SUSE/scf/archive/scf-linux-amd64-1.8.8-pre+cf265.618.gf989f3b.zip
   ```
* Unpack this archive in a directory your choice.
   ```
   mkdir deploy
   cd    deploy
   unzip ../scf-linux-amd64-1.8.8-pre+cf265.618.gf989f3b.zip
   ```
  We now have the helm charts for SCF and UAA in a subdirectory `helm`.
  Additional configuration files are found under `kube`.
  The `scripts` directory contains helpers for cert generation.

* Create custom certs for the deployment by invoking the certification generator:
   ```
   mkdir certs
   ./cert-generator.sh -d ${DOMAIN} -n ${NAMESPACE} -o certs
   ```
  Note: Choosing a different output directory (`certs`) here will require matching changes to the commands deploying the helm charts, below.

* We now have the certificates required by the various components to talk to each other (SCF internals, UAA internals, SCF to UAA).

* Use Helm to deploy UAA. Remember that the previous section gave a reference to the Helm documentation explaining how to install Helm itself. Remember also that in the Vagrant-based setup `helm` is already installed and ready.
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

* Now that SCF is deployed as well its operation can be verified by running the CF smoke and acceptance tests (in this order). This is done via
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
   export STORAGECLASS=persistent

   kubectl get storageclass persistent 2>/dev/null || {
       perl -p -e 's@storage.k8s.io/v1beta1@storage.k8s.io/v1@g' \
           "kube/uaa/kube-test/storage-class-host-path.yml" | \
       kubectl create -f -
   }

   wget https://github.com/SUSE/scf/archive/scf-linux-amd64-1.8.8-pre+cf265.618.gf989f3b.zip

   mkdir deploy
   cd    deploy
   unzip ../scf-linux-amd64-1.8.8-pre+cf265.618.gf989f3b.zip

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

    # Wait for completion (pod-status -w)

    kubectl create \
      --namespace="${NAMESPACE}" \
      --filename="kube/cf/bosh-task/smoke-tests.yml"

    # Wait for completion

    kubectl create \
      --namespace="${NAMESPACE}" \
      --filename="kube/cf/bosh-task/acceptance-tests.yml"
   ```

## Removal/Cleanup via helm

First delete the running system at the kube level

```
    kubectl delete namespace $UAA_NAMESPACE
    kubectl delete namespace $NAMESPACE
```
This will especially remove all the associated volumes as well.

After that use `helm list` to locate the releases for the SCF and UAA charts and `helm delete` to remove them at helm's level as well.

## CF documentation

* https://docs.cloudfoundry.org/
