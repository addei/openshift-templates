# How to install OCP (ABI)

ABI = Agent-based installation

## Files

### SSH key

Generate SSH-key for keybased access to the cluster.

```bash
ssh-keygen
```

### Installation configuration

```yaml install-config.yaml
---
apiVersion: v1
baseDomain: haow.fi
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp-1
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.223.40.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  baremetal:
    apiVIPs:
    - 10.223.40.11
    ingressVIPs:
    - 10.223.40.10
    hosts:
    - name: master-0
      role: master
      bootMACAddress: 08:c0:eb:2d:25:cc
    - name: master-1
      role: master
      bootMACAddress: 08:c0:eb:2d:25:e4
    - name: master-2
      role: master
      bootMACAddress: 08:c0:eb:2d:1f:c4
pullSecret: '<add red hat pull secret here>'
sshKey: 'add ssh key here'

```

### Cluster-node configuration

Node configuration will be defined in the AgentConfig file. This file requires following parametres:

- rendezvousIP
- hosts: this lists nodes and roles for the nodes
  - rootDeviceHints
  - networkConfig

```yaml agent-config.yaml
---
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: ocplab
rendezvousIP: 10.223.40.20
additionalNTPSources: 
- 193.66.253.82
hosts:
  - hostname: master-0
    interfaces:
      - name: ens1f0np0
        macAddress: 08:c0:eb:2d:25:cc
    rootDeviceHints:
      deviceName: /dev/sda
    networkConfig:
      interfaces:
        - name: ens1f0np0.50
          type: vlan
          state: up
          vlan:
            base-iface: ens1f0np0
            id: 50
          ipv4:
            enabled: true
            address:
              - ip: 10.223.40.20
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 194.241.16.66
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.223.40.1
            next-hop-interface: ens1f0np0.50
            table-id: 254
  - hostname: master-1
    interfaces:
      - name: ens1f0np0
        macAddress: 08:c0:eb:2d:25:e4
    rootDeviceHints:
      deviceName: /dev/sda
    networkConfig:
      interfaces:
        - name: ens1f0np0.50
          type: vlan
          state: up
          vlan:
            base-iface: ens1f0np0
            id: 50
          ipv4:
            enabled: true
            address:
              - ip: 10.223.40.21
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 194.241.16.66
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.223.40.1
            next-hop-interface: ens1f0np0.50
            table-id: 254
  - hostname: master-2
    interfaces:
      - name: ens1f0np0
        macAddress: 08:c0:eb:2d:1f:c4
    rootDeviceHints:
      deviceName: /dev/sda
    networkConfig:
      interfaces:
        - name: ens1f0np0.50
          type: vlan
          state: up
          vlan:
            base-iface: ens1f0np0
            id: 50
          ipv4:
            enabled: true
            address:
              - ip: 10.223.40.22
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 194.241.16.66
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.223.40.1
            next-hop-interface: ens1f0np0.50
            table-id: 254

```

### Create ISO-images

This command will use files in the folder for creating correct ISO-images.

```bash
openshift-install --dir . agent create image
```
