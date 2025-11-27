# Lab Lan Installation

This is the installation for my OpenShift Cluster lab.

## Prerequisite

1. Pull secret named `pull-secret`
2. DNS entries for

    ```shell
    api.lab.lan
    api-int.lab.lan
    master1.lab.lan
    *.apps.lab.lan
    ```

## Setup

1. Download agent based installer from [here](https://console.redhat.com/openshift/install/metal/agent-based)
2. Uncompress and untar and move binary to somewhere in your path
3. Install nmstate package `sudo dnf install /usr/bin/nmstatectl -y`

## ISO

### Creating

```shell
git clone git@github.com:jkeam/lab-lan-install.git
cd lab-lan-install
# update agent-config.yaml and install-config.yaml
mkdir working
cp ./agent-config.yaml ./install-config.yaml ./working
openshift-install --dir working agent create image
```

### Flashing

```shell
lsblk  # look for flash drive, assuming /dev/sdb
sudo dd bs=4M if=./working/agent.x86_64.iso of=/dev/sdb status=progress oflag=sync
# wait for done and eject device
```

## Installing

On all machines, boot from flash device.

To track status:

```shell
# log levels: warn, debug, error, info
# bootstrap
openshift-install --dir working agent wait-for bootstrap-complete --log-level=info
# install
openshift-install --dir working agent wait-for install-complete --log-level=info
# debug
# ssh core@<node-ip> agent-gather -O >agent-gather.tar.xz
```

## Adding Nodes

From my linux desktop:

```shell
# grab the worker ign
oc extract -n openshift-machine-api secret/worker-user-data-managed --keys=userData --to=- > worker.ign

# we already created the new-worker.ign
cat ./workers/new-worker.ign

# download the right version of the linux installer
curl -k "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.19/openshift-install-linux.tar.gz" > openshift-install-linux.tar.gz

# download ISO
ISO_URL=$(./openshift-install coreos print-stream-json | grep location | grep x86_64 | grep iso | cut -d\" -f4)
curl -L $ISO_URL -o rhcos-live.iso

# same as before
# plug flash drive in
lsblk  # look for flash drive
sudo dd bs=4M if=./rhcos-live.iso of=/dev/sde status=progress oflag=sync

# upload ignition files to local webserver
#   assuming url is: https://httpd-server-cluster-services.apps.lab.lan/worker.ign
#   assuming url is: https://httpd-server-cluster-services.apps.lab.lan/new-worker.ign
```

Plug flash into new node and boot from it.

```shell
# apply static ip
nmcli con mod "Wired connection 1" ipv4.method manual /
    ipv4.addresses 192.168.1.203 /
    ipv4.gateway 192.168.1.1 /
    ipv4.dns 192.169.1.201 /
    802-3-ethernet.mtu 9000

# restart
nmcli con down "Wired connection 1"
nmcli con up "Wired connection 1"
```

## Cert Manager

1. Install cert-manager operator into the default namespace.
2. Run the following:

    ```shell
    oc new-project cert-manager
    oc apply -k ./cert-manager
    ```

3. Create cert for app, do this for every app.

    ```shell
    oc project chatbot
    # Optionally update the values in manifest
    oc apply -f ./cert-manager/certificate.yaml
    # Wait a bit for the dns propagation to take place,
    #   after secret named app-cert-tls-secret appears then apply below
    # Update the values <replace-me> in manifest
    #   as well as keam.org with your domain name
    oc apply -f ./cert-manager/route.yaml
    ```

## AMD ROCm

### Installation

1. Install [NFD](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/specialized_hardware_and_driver_enablement/psap-node-feature-discovery-operator)
2. Create NodeFeatureDiscovery object like [this](https://instinct.docs.amd.com/projects/gpu-operator/en/release-v1.4.0/installation/openshift-olm.html#create-node-feature-discovery-rule)
3. Install [KMM](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/specialized_hardware_and_driver_enablement/kernel-module-management-operator), not the Hub one
4. Install AMD GPU Operator, can use the latest below or use the `Certified`
version from the Operator Hub by searching for `amd`

    ```shell
    # Add helm chart
    helm repo add rocm https://rocm.github.io/gpu-operator
    helm repo update

    # Install operator
    helm install amd-gpu-operator rocm/gpu-operator-charts \
      --namespace kube-amd-gpu \
      --create-namespace \
      --version=v1.4.0

    # DeviceConfig custom resource is automatically created, to edit:
    kubectl edit deviceconfigs -n kube-amd-gpu default
    ```

### Architecture

![AMD Architecture](https://instinct.docs.amd.com/projects/gpu-operator/en/release-v1.4.0/_images/amd-gpu-operator-diagram.png)

## Links

1. [NMState](https://access.redhat.com/solutions/7020319)
2. [Agent Based Installer](https://console.redhat.com/openshift/install/metal/agent-based)
3. [Bare metal install docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_an_on-premise_cluster_with_the_agent-based_installer/index)
4. [Bare metal install troubleshooting docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-basic#installing-ocp-agent-gather-log_installing-with-agent-basic)
5. [Adding bare metal worker nodes to SNO](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html-single/nodes/index#nodes-sno-worker-nodes)
6. [AMD GPU Operator](https://github.com/ROCm/gpu-operator)
7. [AMD GPU Docs](https://instinct.docs.amd.com/projects/gpu-operator/en/release-v1.4.0/overview.html)
