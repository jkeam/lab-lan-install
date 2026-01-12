# Lab Lan Installation

This is the installation for my OpenShift Cluster lab.

## Prerequisite

1. Pull secret named `pull-secret`
2. DNS entries for

    ```shell
    api.lab.keam.org
    api-int.lab.keam.org
    master1.lab.keam.org
    *.apps.lab.keam.org

    # Make sure there is no entry for *.lab.keam.org as that is known to cause issues.
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

## GitHub OAuth

1. Create GitHub Organization and update `./github/oauth.yaml`

2. Create GitHub OAuth app with info and get the generated Client Id and Client Secret

    ```properties
    name: whatever you want
    homepage_url: https://oauth-openshift.apps.<cluster-name>.<cluster-domain>.
    callback_url: https://oauth-openshift.apps.<cluster-name>.<cluster-domain>/oauth2callback/github
    ```

3. Take those values and update
`./github/oauth.yaml` and `./github/github-oauth-secret.yaml` respectively

4. Create the secret

    ```shell
    oc apply -f ./github/github-oauth-secret.yaml
    ```

5. Edit OpenShift OAuth

    ```shell
    oc edit oauth/cluster  # add in the data from ./github/oauth.yaml
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

```shell
# create digital ocean api key
oc apply -f ./cert-manager/secret.yaml  # replace key

# hook into let's encrypt using the digital ocean dns challenge
oc apply -f ./cert-manager/issuer.yaml

# create api cert, wait 2 min to create
oc apply -f ./cert-manager/api-cert.yaml

# create apps cert, wait 2 min to create
oc apply -f ./cert-manager/router-cert.yaml

# patch api
oc patch apiserver cluster --type merge --patch="{\"spec\": {\"servingCerts\": {\"namedCertificates\": [ { \"names\": [  \"api.lab.keam.org\"  ], \"servingCertificate\": {\"name\": \"api-certs\" }}]}}}" --insecure-skip-tls-verify

# patch ingress controller for apps
oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"spec": { "defaultCertificate": { "name": "router-certs" }}}' --insecure-skip-tls-verify
```

## Garbage Collection

Update GC to be a bit more aggressive in cleaning up disk.

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: gc-master-kubeconfig
spec:
  machineConfigPoolSelector:
    matchLabels:
      pools.operator.machineconfiguration.openshift.io/master: ""
  kubeletConfig:
    evictionPressureTransitionPeriod: 3m
    imageMinimumGCAge: 5m            # default is 2m
    imageGCHighThresholdPercent: 70  # default is 85
    imageGCLowThresholdPercent: 50   # default is 80
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
8. [GitHub Auth](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/authentication_and_authorization/configuring-identity-providers#configuring-github-identity-provider)
9. [HTPasswd Auth](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/authentication_and_authorization/configuring-identity-providers#configuring-htpasswd-identity-provider)
10. [TLS Routes](https://medium.com/@evgeniy.phv/using-cert-manager-in-openshift-okd-part-2-configuration-and-ssl-certificate-installation-fd5a86b734df)
11. [Node GC](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/nodes/working-with-nodes#nodes-nodes-garbage-collection)
