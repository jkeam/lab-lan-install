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

## Links

1. [NMState](https://access.redhat.com/solutions/7020319)
2. [Agent Based Installer](https://console.redhat.com/openshift/install/metal/agent-based)
3. [Bare metal install docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_an_on-premise_cluster_with_the_agent-based_installer/index)
4. [Bare metal install troubleshooting docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-basic#installing-ocp-agent-gather-log_installing-with-agent-basic)
5. [Adding bare metal worker nodes to SNO](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html-single/nodes/index#nodes-sno-worker-nodes)
