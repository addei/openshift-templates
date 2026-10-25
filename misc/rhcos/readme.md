# RHCOS OCP ABI machine

This document provides guidance on customizing and deploying Red Hat CoreOS (RHCOS) virtual machines for OpenShift Agent-Based Installer (ABI) environments. It introduces Butane as a tool for generating Ignition configs for various platforms, explains how to modify RHCOS images (such as adding users and agents), and highlights the need for platform-specific guest agents (e.g., open-vm-tools for VMware). The guide also points to resources for getting started and customizing your own RHCOS VM images.

## Getting started with Butane

Read and also understand the base princible of Butane, and why is it used and how it is used.

[Butane - Getting started](https://coreos.github.io/butane/getting-started/)

Note, even though documentation only shows ```fcos``` as supported system, Butane is used for also generating wide variety of different ignition files for initializing and booting up CoreOS systems, like ```openshift``` (aka rhcos), ```flatcar```, ```r4e``` and ```fiot```.

```rhcos``` was initially also and CoreOS variant and also an option, but was later removed and replaced by ```openshift``` as Red Hat CoreOS product was dedicated for ONLY as OS for Red Hat Openshift platform. 

[From rhcos Version 0.1.0 to openshift Version 4.8.0](https://coreos.github.io/butane/upgrading-openshift/#from-rhcos-version-010-to-openshift-version-480)

As noted in the link above, by default ```butane``` creates [```MachineConfig```](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/machine_apis/machineconfig-machineconfiguration-openshift-io-v1)

But this can be overriden by ```-r``` or ```--raw``` flags on execution of the Butane command, which makes it then output  clear ```ignition``` files that can be injected to the base OS image.

This means, you can also still create rhcos systems.

## Customize image for your own liking

Please see [```rhcos-butane.bu```](misc/rhcos/rhcos-butane.bu)

Note in this example, RHCOS server is only modified to create system user ```admin```, set the user be part of the wheel group and inject SSH public key for remote access. Also qemu-guest-agent is installed, but other hosting solutions (like VMware or else) require their own agents (like [```open-vm-tools```](https://github.com/vmware/open-vm-tools) for VMware platforms). For baremetal setup, systemd unit file can be removed completely.

## How to create Red Hat CoreOS VM

### Download ISO

Create iso-folder
```bash
mkdir -p iso
```
Download ISO image of rhcos using ```curl```
```bash
curl -o iso/rhcos-4.19.10-x86_64-live-iso.x86_64.iso https://mirror.openshift.com/pub/openshift-v4/amd64/dependencies/rhcos/latest/rhcos-4.19.10-x86_64-live-iso.x86_64.iso
```
or with ```wget```

```bash
wget -O iso/rhcos-4.19.10-x86_64-live-iso.x86_64.iso https://mirror.openshift.com/pub/openshift-v4/amd64/dependencies/rhcos/latest/rhcos-4.19.10-x86_64-live-iso.x86_64.iso
```

### Pull tools using podman

Pull tools from quay.io

```bash
podman pull quay.io/coreos/butane
podman pull quay.io/coreos/coreos-installer
podman pull quay.io/coreos/ignition-validate
```

Create aliases for people who can not understand containers

```bash
alias butane='podman run --rm --interactive         \
              --security-opt label=disable          \
              --volume "${PWD}":/pwd --workdir /pwd \
              quay.io/coreos/butane:release'
alias ignition-validate='podman run --rm --interactive         \
              --security-opt label=disable          \
              --volume "${PWD}":/pwd --workdir /pwd \
              quay.io/coreos/ignition-validate:release'
```

### Creating ignition files from butane files

```bash
butane --pretty --strict "./rhcos.bu" --output "rhcos.ign"
```
Note, butane has for openshift variant hardcoded check if any other user than core is present in the file.
You can also validate ignition files using ```ignition-validate``` command!

```bash
ignition-validate "rhcos.bu"
```

### Injecting ignition files using ```coreos-installer```

```bash
sudo podman run --pull=always --privileged --rm \
    -v /dev:/dev -v /run/udev:/run/udev -v .:/data -w /data \
    quay.io/coreos/coreos-installer:release \
    iso customize \
    --dest-ignition "rhcos.ign" \
    --dest-device "/dev/vda" \
    -o "./iso/rhcos-4.19.10-x86_64.iso" \
    "./iso/rhcos-4.19.10-x86_64-live-iso.x86_64.iso"
```

## Links

Butane API-reference