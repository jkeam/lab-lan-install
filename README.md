# Lab Lan Installation

This is the installation for my OpenShift Cluster lab.

## Prerequisite

1. Pull secret named `pull-secret`

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

## Links

1. [NMState](https://access.redhat.com/solutions/7020319)
2. [Agent Based Installer](https://console.redhat.com/openshift/install/metal/agent-based)
3. [Bare metal install docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_an_on-premise_cluster_with_the_agent-based_installer/index)
4. [Bare metal install troubleshooting docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-basic#installing-ocp-agent-gather-log_installing-with-agent-basic)
