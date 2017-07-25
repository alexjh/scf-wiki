## Requirements for Kube

The various machines (`api`, `kube`, and `node`) of the k8s cluster must be configured in a particular way to support the execution of `SCF`. These requirements are, in general:

* Kernel parameters `cgroup_enable=memory` and `swapaccount=1`
* `docker info` must show `overlay2`.
* `kube-dns` must be be running and `4/4 ready`.
* Either `ntp` or `systemd-timesyncd` must be installed and active.
* The k8s cluster must have a storage class SCF can refer to.
* Privileged must be enabled in `kube-apiserver`.
* Privileged must be enabled in `kubelet`.
* DNS has to resolve the domain stored in the environment variable `SCF_DOMAIN`.
* The `systemd` setting `TasksMax` must be set to infinity.
* Helm's Tiller has to be installed and active.

|Category|Explanation|
|---|---|
|api| Requirements on the hosts for the k8s master nodes (running `apiserver`) |
|kube| Requirements of the cluster itself, via `kubectl` |
|node| Requirements on the hosts for the k8s worker nodes (running `kubelet`) |

An easy way of setting up a small single-machine k8s cluster with all the necessary properties is to use the Vagrant definition in the SCF repository. The details of this approach are explained in https://github.com/SUSE/scf/blob/develop/README.md#deploying-scf-on-vagrant

## Kube verification

For ease of verification of these requirements a script (`k8s-ready-state-check.sh`) is made available which contains the necessary checks.

To get help invoke this script via
```
k8s-ready-state-check.sh -v
```
This will especially note the various machine categories. When invoked with the name of machine category (`api`, `kube`, and `node`), i.e. like
```
k8s-ready-state-check.sh kube
```
the script will run the tests applicable to the named category.
Positive results are prefixed with `Verified: `,
whereas failed requirements are prefixed with `Configuration problem detected:`.

## SUSE UI Backlinks
## Helm installation reference

SCF uses Helm charts to deploy on k8s clusters.
To install Helm see

* https://docs.helm.sh/using_helm/#quickstart

## Installation via helm (includes cert gen)

Consider the following an annotated session demonstrating how to deploy SCF/UAA on k8s. In other words, explanations interspersed with example commands and vice versa.

To install SCF
* __Choose__ the `DOMAIN` of SCF and use standard tools to set up the DNS
  so the IP address where SCF is exposed is reachable under `*.DOMAIN`.
   ```
   export DOMAIN=cf-dev.io
   ```
* __Choose__ the `NAMESPACE` of SCF, i.e. the k8s namespace the SCF components will run in later.
   ```
   export NAMESPACE=scf
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

* Save all choices to environment variables.
  These are used in the coming commands.
* Get the distribution archive from **XXX**
  ```
  wget XXX/scf-linux-amd64-1.8.8-pre+cf265.618.gf989f3b.zip
  ```
* Unpack this archive in a directory your choice.
  ```
  mkdir deploy
  cd    deploy
  unzip ../scf-linux-amd64-1.8.8-pre+cf265.618.gf989f3b.zip
  ```
  We now have the helm charts for SCF and UAA in a subdirectory `helm`.
  Additional k8s configuration files are found under `kube`.
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
     --namespace ${NAMESPACE} \
     --values certs/uaa-cert-values.yaml \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}"
  ```

* With UAA deployed, use Helm to deploy SCF.
  ```
  helm install helm/cf \
     --namespace ${NAMESPACE} \
     --values certs/scf-cert-values.yaml \
     --set "env.CLUSTER_ADMIN_PASSWORD=$CLUSTER_ADMIN_PASSWORD" \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}" \
     --set "env.UAA_HOST=${UAA_HOST}" \
     --set "env.UAA_PORT=${UAA_PORT}"
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

## Point to CF docs

* https://docs.cloudfoundry.org/
