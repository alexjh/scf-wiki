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

To install SCF
* Choose the `DOMAIN` of SCF and use standard tools to set up the DNS
  so the IP address where SCF is exposed is reachable under `*.DOMAIN`.

* Choose the `NAMESPACE` of SCF, i.e. the k8s namespace the SCF components will run in later.

* Save both choices to environment variables of the same name.
  These are used in the coming commands.

* Get the distribution archive from **XXX**
* Unpack this archive in a directory your choice. In the following discussion this directory is called `DA`. Replace its uses with the name of your own choice.

* Invoke the certification generator via
  ```
   cd DA
   cert-generator.sh -d ${DOMAIN} -n ${NAMESPACE} -o helm
  ```
  By default the results are written into the current directory.
  If that does not suit use `-o` to change that location. The directory must exist.
  (**XXX** We may have to set it here to match other places expectations, i.e. helm).

* We now have the certificates required by the SCF internals to talk to each other.

* Use Helm to deploy it, like so
  ```
  cd DA/helm
  helm install helm \
     --namespace ${NAMESPACE} \
     --set "env.CLUSTER_ADMIN_PASSWORD=$CLUSTER_ADMIN_PASSWORD" \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}" \
     --set "env.UAA_HOST=${UAA_HOST}" \
     --set "env.UAA_PORT=${UAA_PORT}"

     XXX the location of the charts and certs is missing here...
  ```
  The previous section gave a reference to the Helm documentation explaining how to install Helm itself.

   The first two variables specify domain for the SCF api endpoint, and the password for the SCF administrator.
   The remainder provide the access key and location of the UAA/Oauth2 server SCF should use.
 
* Now that SCF is deployed it can be verified by running the smoke and CF acceptance tests (in this order). This is done via

   ```
   kubectl create --namespace="${NAMESPACE}" --filename="DA/cf/bosh-task/smoke-tests.yml"
   ```
   and
   ```
   kubectl create --namespace="${NAMESPACE}" --filename="DA/cf/bosh-task/acceptance-tests.yml"
   ```

## Point to CF docs

* https://docs.cloudfoundry.org/
