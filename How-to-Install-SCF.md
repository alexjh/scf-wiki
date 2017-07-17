## Requirements for Kube (point to Vagrant for non-kube-can-setup-people)

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
## SUSE UI Backlinks
## Helm installation reference

* https://docs.helm.sh/using_helm/#quickstart

## Installation via helm (includes cert gen)
## Point to CF docs

* https://docs.cloudfoundry.org/
