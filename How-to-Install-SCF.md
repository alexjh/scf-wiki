## Requirements for Kube (point to Vagrant for non-kube-can-setup-people)

The various machines (`api`, `kube`, and `node`) used to run the k8s cluster must be configured in a particular way to support the execution of SCF. These requirements are, in general:

* `cgroup_enable` memory
* `swapaccount` enable
* `docker info` must show `overlay2`
* `kube-dns` must be be running and `4/4 ready`
* Either `ntp` or `systemd-timesyncd` must be installed and active
* A storage class has to exist in K8s
* Privileged must be enabled in `kube-apiserver`
* Privileged must be enabled in `kubelet`
* DNS has to resolve the `SCF_DOMAIN`.
* `systemd` setting `TasksMax` must be set to infinity

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

* https://docs.helm.sh/using_helm/#quickstart

## Installation via helm (includes cert gen)

Topic to fill
* download the "scf" package
* unpack it
* use the cert generation script
* setup DNS for cloud foundry (*.DOMAIN pointing to the IP where CF is exposed)
* use the helm cli to deploy scf
* verify the deployment




## Point to CF docs

* https://docs.cloudfoundry.org/
