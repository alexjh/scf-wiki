## Requirements for Kube (point to Vagrant for non-kube-can-setup-people)

The various machines (`api`, `kube`, and `node`) of the k8s cluster must be configured in a particular way to support the execution of `SCF`. These requirements are, in general:

* `cgroup_enable` memory.
* `swapaccount` enable.
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
* Get the distribution archive from **XXX**
* Unpack this archive in a directory your choice.
* Generate the certificates required by the SCF internals to talk to each other.
  This is done by invoking **XXX** coming with the distribution, like so

  ```
  XXX
  ```

* Choose the DOMAIN of SCF and use standard tools to set up the DNS so the IP address where SCF is exposed is reachable under `*.DOMAIN`.
* Use Helm to deploy it, like so
  ```
  helm install helm \
     --namespace ${NAMESPACE} \
     --set "env.CLUSTER_ADMIN_PASSWORD=$CLUSTER_ADMIN_PASSWORD" \
     --set "env.DOMAIN=${DOMAIN}" \
     --set "env.UAA_ADMIN_CLIENT_SECRET=${UAA_ADMIN_CLIENT_SECRET}" \
     --set "env.UAA_HOST=${UAA_HOST}" \
     --set "env.UAA_PORT=${UAA_PORT}"
  ```
  The previous section gave a reference to the Helm documentation explaining how to install Helm itself.

   The first two variables specify domain for the SCF api endpoint, and the password for the SCF administrator.
   The remainder provide the access key and location of the UAA/Oauth2 server SCF should use.
 
* Now that SCF is deployed it can be verified by running the smoke and CF acceptance tests (in this order). This is done via

   ```
   kubectl create --namespace="${NAMESPACE}" --filename="XXX/smoke-tests.yml"
   ```
   and
   ```
   kubectl create --namespace="${NAMESPACE}" --filename="XXX/acceptance-tests.yml"
   ```

## Point to CF docs

* https://docs.cloudfoundry.org/
