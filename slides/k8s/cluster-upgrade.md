# Upgrading clusters

- It's *recommended* to run consistent versions across a cluster

  (mostly to have feature parity and latest security updates)

- It's not *mandatory*

  (otherwise, cluster upgrades would be a nightmare!)

- Components can be upgraded one at a time without problems

<!-- ##VERSION## -->

---

## Checking what we're running

- It's easy to check the version for the API server

.lab[

- Log into node `oldversion1`

- Check the version of kubectl and of the API server:
  ```bash
  kubectl version
  ```

]

- In a HA setup with multiple API servers, they can have different versions

- Running the command above multiple times can return different values

---

## Node versions

- It's also easy to check the version of kubelet

.lab[

- Check node versions (includes kubelet, kernel, container engine):
  ```bash
  kubectl get nodes -o wide
  ```

]

- Different nodes can run different kubelet versions

- Different nodes can run different kernel versions

- Different nodes can run different container engines

---

## Control plane versions

- If the control plane is self-hosted (running in pods), we can check it

.lab[

- Show image versions for all pods in `kube-system` namespace:
  ```bash
    kubectl --namespace=kube-system get pods -o json \
            | jq -r '
              .items[]
              | [.spec.nodeName, .metadata.name]
                + 
                (.spec.containers[].image | split(":"))
              | @tsv
              ' \
            | column -t
  ```

]

---

## What version are we running anyway?

- When I say, "I'm running Kubernetes 1.22", is that the version of:

  - kubectl

  - API server

  - kubelet

  - controller manager

  - something else?

---

## Other versions that are important

- etcd

- kube-dns or CoreDNS

- CNI plugin(s)

- Network controller, network policy controller

- Container engine

- Linux kernel

---

## Important questions

- Should we upgrade the control plane before or after the kubelets?

- Within the control plane, should we upgrade the API server first or last?

- How often should we upgrade?

- How long are versions maintained?

- All the answers are in [the documentation about version skew policy](https://kubernetes.io/docs/setup/release/version-skew-policy/)!

- Let's review the key elements together ...

---

## Kubernetes uses semantic versioning

- Kubernetes versions look like MAJOR.MINOR.PATCH; e.g. in 1.22.17:

  - MAJOR = 1
  - MINOR = 22
  - PATCH = 17

- It's always possible to mix and match different PATCH releases

  (e.g. 1.22.17 and 1.22.5 are compatible)

- It is recommended to run the latest PATCH release

  (but it's mandatory only when there is a security advisory)

---

## Version skew

- API server must be more recent than its clients (kubelet and control plane)

- ... Which means it must always be upgraded first

- All components support a difference of one¹ MINOR version

- This allows live upgrades (since we can mix e.g. 1.22 and 1.23)

- It also means that going from 1.22 to 1.24 requires going through 1.23

.footnote[¹Except kubelet, which can be up to two MINOR behind API server,
and kubectl, which can be one MINOR ahead or behind API server.]

---

## Release cycle

- There is a new PATCH relese whenever necessary

  (every few weeks, or "ASAP" when there is a security vulnerability)

- There is a new MINOR release every 3 months (approximately)

- At any given time, three MINOR releases are maintained

- ... Which means that MINOR releases are maintained approximately 9 months

- We should expect to upgrade at least every 3 months (on average)

---

## General guidelines

- To update a component, use whatever was used to install it

- If it's a distro package, update that distro package

- If it's a container or pod, update that container or pod

- If you used configuration management, update with that

---

## Know where your binaries come from

- Sometimes, we need to upgrade *quickly*

  (when a vulnerability is announced and patched)

- If we are using an installer, we should:

  - make sure it's using upstream packages

  - or make sure that whatever packages it uses are current

  - make sure we can tell it to pin specific component versions

---

## In practice

- We are going to update a few cluster components

- We will change the kubelet version on one node

- We will change the version of the API server

- We will work with cluster `oldversion` (nodes `oldversion1`, `oldversion2`, `oldversion3`)

---

## Updating the API server

- This cluster has been deployed with kubeadm

- The control plane runs in *static pods*

- These pods are started automatically by kubelet

  (even when kubelet can't contact the API server)

- They are defined in YAML files in `/etc/kubernetes/manifests`

  (this path is set by a kubelet command-line flag)

- kubelet automatically updates the pods when the files are changed

---

## Changing the API server version

- We will edit the YAML file to use a different image version

.lab[

- Log into node `oldversion1`

- Check API server version:
  ```bash
  kubectl version
  ```

- Edit the API server pod manifest:
  ```bash
  sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
  ```

- Look for the `image:` line, and update it to e.g. `v1.24.1`

]

---

## Checking what we've done

- The API server will be briefly unavailable while kubelet restarts it

.lab[

- Check the API server version:
  ```bash
  kubectl version
  ```

]

---

## Was that a good idea?

--

**No!**

--

- Remember the guideline we gave earlier:

  *To update a component, use whatever was used to install it.*

- This control plane was deployed with kubeadm

- We should use kubeadm to upgrade it!

---

## Updating the whole control plane

- Let's make it right, and use kubeadm to upgrade the entire control plane

  (note: this is possible only because the cluster was installed with kubeadm)

.lab[

- Check what will be upgraded:
  ```bash
  sudo kubeadm upgrade plan
  ```

]

Note 1: kubeadm thinks that our cluster is running 1.24.1.
<br/>It is confused by our manual upgrade of the API server!

Note 2: kubeadm itself is still version 1.22.1..
<br/>It doesn't know how to upgrade do 1.23.X.

---

## Upgrading kubeadm

- First things first: we need to upgrade kubeadm

.lab[

- Upgrade kubeadm:
  ```
  sudo apt install kubeadm=1.27.0-00
  ```

- Check what kubeadm tells us:
  ```
  sudo kubeadm upgrade plan
  ```

]

Problem: kubeadm doesn't know know how to handle
upgrades from version 1.22.

This is because we installed version 1.27.

We need to install kubeadm version 1.23.X.

---

## Downgrading kubeadm

- We need to go back to kubeadm version 1.23.X.

.lab[

- View available versions for package `kubeadm`:
  ```bash
  apt show kubeadm -a | grep ^Version | grep 1.23
  ```

- Downgrade kubeadm:
  ```
  sudo apt install kubeadm=1.23.0-00
  ```

- Check what kubeadm tells us:
  ```
  sudo kubeadm upgrade plan
  ```

]

kubeadm should now agree to upgrade to 1.23.X.

---

## Reverting our manual API server upgrade

- First, we should revert our `image:` change

  (so that kubeadm executes the right migration steps)

.lab[

- Edit the API server pod manifest:
  ```bash
  sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
  ```

- Look for the `image:` line, and restore it to the original value

  (e.g. `v1.22.17`)

- Wait for the control plane to come back up

]

---

## Upgrading the cluster with kubeadm

- Now we can let kubeadm do its job!

.lab[

- Perform the upgrade:
  ```bash
  sudo kubeadm upgrade apply v1.23.0
  ```

]

---

## Updating kubelet

- These nodes have been installed using the official Kubernetes packages

- We can therefore use `apt` or `apt-get`

.lab[

- Log into node `oldversion2`

- View available versions for package `kubelet`:
  ```bash
  apt show kubelet -a | grep ^Version
  ```

- Upgrade kubelet:
  ```bash
  sudo apt install kubelet=1.23.0-00
  ```

]

---

## Checking what we've done

.lab[

- Log into node `oldversion1`

- Check node versions:
  ```bash
  kubectl get nodes -o wide
  ```

- Create a deployment and scale it to make sure that the node still works

]

---

## Was that a good idea?

--

**Almost!**

--

- Yes, kubelet was installed with distribution packages

- However, kubeadm took care of configuring kubelet

  (when doing `kubeadm join ...`)

- We were supposed to run a special command *before* upgrading kubelet!

- That command should be executed on each node

- It will download the kubelet configuration generated by kubeadm

---

## Upgrading kubelet the right way

- We need to upgrade kubeadm, upgrade kubelet config, then upgrade kubelet

  (after upgrading the control plane)

.lab[

- Download the configuration on each node, and upgrade kubelet:
  ```bash
    for N in 1 2 3; do
      ssh oldversion$N "
        sudo apt install kubeadm=1.23.0-00 &&
        sudo kubeadm upgrade node &&
        sudo apt install kubelet=1.23.0-00"
    done
  ```
]

---

## Checking what we've done

- All our nodes should now be updated to version 1.23.0

.lab[

- Check nodes versions:
  ```bash
  kubectl get nodes -o wide
  ```

]

---

class: extra-details

## Skipping versions

- This example worked because we went from 1.22 to 1.23

- If you are upgrading from e.g. 1.21, you will have to go through 1.22 first

- This means upgrading kubeadm to 1.22.X, then using it to upgrade the cluster

- Then upgrading kubeadm to 1.23.X, etc.

- **Make sure to read the release notes before upgrading!**

???

:EN:- Best practices for cluster upgrades
:EN:- Example: upgrading a kubeadm cluster

:FR:- Bonnes pratiques pour la mise à jour des clusters
:FR:- Exemple : mettre à jour un cluster kubeadm
