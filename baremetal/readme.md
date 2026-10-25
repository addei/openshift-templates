# Compact OpenShift Cluster Deployment (Agent-Based Install)

## 1. Prerequisites

- **Nodes:**
	- 3 master nodes (control plane)
		- Each: 4 vCPU, 16GB RAM
		- Bootstrap master (rendezvousIP machine): 32GB RAM
	- All nodes must support virtualization and have at least 120GB disk space.

- **Network:**  
	- Dynamic (via DHCP reservation) or static IPs for all nodes
	- Two VIP addresses (API + API-int, Ingress)
	- Working NTP server(s)  
	- DNS server (e.g., Pi-hole, BIND)
	- HAProxy load balancer

- **Software:**
	- `openshift-install` binary
	- SSH key for node access
	- Red Hat pull secret (for OpenShift images)
    - OKD disables this.

---

## 2. VIP Addresses & HAProxy

Assign three VIPs in your machine network (e.g., `192.168.1.10`, `192.168.1.11`):

- **API VIP:** `192.168.1.11` (port 6443)
- **Ingress VIP:** `192.168.1.10` (ports 80, 443)

Note! Machine Config Server also uses API VIP, but requires different port `22623`
- **Machine Config Server:** `192.168.1.11` (port 22623)

### HAProxy Example Configuration

```haproxy
frontend api
		bind 192.168.1.11:6443
		default_backend api_servers

backend api_servers
		balance roundrobin
		server master-0 192.168.0.20:6443 check
		server master-1 192.168.0.21:6443 check
		server master-2 192.168.0.22:6443 check

frontend ingress_http
		bind 192.168.1.10:80
		default_backend ingress_servers

frontend ingress_https
		bind 192.168.1.10:443
		default_backend ingress_servers

backend ingress_servers
		balance roundrobin
		server master-0 192.168.0.20:443 check
		server master-1 192.168.0.21:443 check
		server master-2 192.168.0.22:443 check

frontend mcs
		bind 192.168.1.11:22623
		default_backend mcs_servers

backend mcs_servers
		balance roundrobin
		server master-0 192.168.0.20:22623 check
		server master-1 192.168.0.21:22623 check
		server master-2 192.168.0.22:22623 check
```

---

## 3. DNS Records (Type A)

Create the following A-records in your DNS server:

| Name                                   | IP Address         | Purpose                |
|---------------------------------------- |------------------- |----------------------- |
| api.ocp-1.example.com                  | 192.168.1.11       | API VIP                |
| api-int.ocp-1.example.com              | 192.168.1.11       | Internal API VIP       |
| *.apps.ocp-1.example.com               | 192.168.1.10       | Ingress VIP            |
| master-0.ocp-1.example.com             | 192.168.0.20       | Node 1                 |
| master-1.ocp-1.example.com             | 192.168.0.21       | Node 2                 |
| master-2.ocp-1.example.com             | 192.168.0.22       | Node 3                 |

---

## 4. NTP Configuration

Ensure all nodes can reach public NTP servers or a local NTP server.  
Example NTP sources:
- `0.pool.ntp.org`
- `1.pool.ntp.org`
- `2.pool.ntp.org`

Configure in your agent config:
```yaml
additionalNTPSources: 
- 0.pool.ntp.org
- 1.pool.ntp.org
- 2.pool.ntp.org
```

---

## 5. Creating install-config.yaml

Example (`baremetal/compact/access/install-config.yaml`):

```yaml
apiVersion: v1
baseDomain: example.com
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
	- cidr: 192.168.0.0/24
	networkType: OVNKubernetes
	serviceNetwork:
	- 172.30.0.0/16
platform:
	baremetal:
		apiVIPs:
		- 192.168.1.11
		ingressVIPs:
		- 192.168.1.10
		hosts:
		- name: master-0
			role: master
			bootMACAddress: 52:54:00:12:34:5a
		- name: master-1
			role: master
			bootMACAddress: 52:54:00:65:43:21
		- name: master-2
			role: master
			bootMACAddress: 52:54:00:ab:cd:ef
pullSecret: '<add red hat pull secret here>'
sshKey: '<add ssh key here>'
```

Note! Deployment can be done also without baremetal definations. Customer is then responsible for the deployment of the cluster deployment.

---

## 6. Creating agent-config.yaml

Example (`baremetal/compact/access/agent-config.yaml`):

```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
	name: ocp-1
rendezvousIP: 192.168.0.20
additionalNTPSources: 
- 0.pool.ntp.org
- 1.pool.ntp.org
- 2.pool.ntp.org
hosts:
	- hostname: master-0
		interfaces:
			- name: eno1
				macAddress: 52:54:00:12:34:5a
		rootDeviceHints:
			deviceName: /dev/sda
		networkConfig:
			interfaces:
				- name: eno1
					type: ethernet
					state: up
					ipv4:
						enabled: true
						address:
							- ip: 192.168.0.20
								prefix-length: 24
						dhcp: false
			dns-resolver:
				config:
					server:
						- 1.1.1.1
			routes:
				config:
					- destination: 0.0.0.0/0
						next-hop-address: 192.168.0.1
						next-hop-interface: eno1
						table-id: 254
	# Repeat for master-1 and master-2
```

NOTE: ```rootDeviceHints deviceName``` and ```networkConfig interfaces name``` needs to reflect how server sees the underlying hardware. This can be checked by installing temporary Fedora/Centos based Linux server and checking our ```lsblk``` and ```ip -c link``` command output.

---

## 7. Backup Before Creating Agent ISO

Before creating the agent ISO, backup your configuration and data:

```bash
mkdir -p okd-install
mkdir -p okd-install-backup
cp -r okd-install/ okd-install-backup/
```

---

## 8. Download required tools

### 1. Go to the Official OpenShift Downloads Page

Visit:  
[https://console.redhat.com/openshift/downloads](https://console.redhat.com/openshift/downloads)

> You need a Red Hat account to access the downloads.

---

### 2. Download the Binaries

- **oc CLI:**  
  - Select your platform (Linux, macOS, Windows).
  - Download the "OpenShift CLI (`oc`)" archive.

- **openshift-install:**  
  - Select your platform.
  - Download the "OpenShift Installer" archive.

---

### 3. Extract and Install

For Linux/macOS:
```bash
tar -xvf openshift-client-*.tar.gz
sudo mv oc kubectl /usr/local/bin/

tar -xvf openshift-install-*.tar.gz
sudo mv openshift-install /usr/local/bin/
```

This installs openshift client and install binaries for the underlying system.

## 9. Create Agent ISO

Run the following command to generate the agent ISO:
```bash
openshift-install agent create image --dir okd-install/
```

Note! ```openshift-install``` requires ```nmstatectl``` for validating network configuration!

## 10. Upload ISO images to servers

Use your tooling for uploading ISO to cluster nodes (baremetal or VM).

## 11. VM Preparation Checklist for Agent-Based OpenShift Install

1. **Resources**
   - CPU: Minimum 4 vCPU per node (32GB RAM for bootstrap/rendezvous node, 16GB RAM for others)
   - Disk: At least 120GB per node (note installation tests disk speed also!)

2. **MAC Address**
   - Each VM must have a unique, static MAC address matching the one in your `agent-config.yaml` and `install-config.yaml`.

3. **Networking**
   - Static IP addresses (no DHCP unless explicitly configured)
   - Correct subnet and gateway
   - Network connectivity between all nodes, VIPs, and the installer
   - No overlapping with Docker or other bridge networks
   - DNS resolution for all required records (API, Ingress, nodes)
   - NTP access for time synchronization
   - Note! Installation process also tests network speed and latency.

4. **Boot Order**
   - Ensure VMs boot from the agent ISO during installation.

5. **Virtualization Settings**
   - Enable virtualization extensions (VT-x/AMD-V)
   - Disable Secure Boot if it interferes with booting the ISO

6. **Access**
   - SSH access for troubleshooting (if needed)
    - You can access the node using SSH to node IP-address with user `core` and the ssh-key used for the cluster deployment.

7. **Other**
   - Correct root device specified in `agent-config.yaml`
   - Proper network interface name (e.g., `eno1`, `eth0`) matching config
   - Sufficient bandwidth and low latency between nodes

**Summary:**  
Check resources, MAC, networking, boot order, virtualization, access, and config alignment for a smooth agent-based install.

## 12. Start installation
Once all prerequisites are met and the agent ISO is created and distributed to your nodes, you can begin the OpenShift installation process:

### Steps to Start Installation

1. **Boot all nodes from the agent ISO**
	- Ensure each node is configured to boot from the ISO you generated in the previous steps.
	- The rendezvous node (bootstrap master) should be powered on first, followed by the other master nodes.

2. **Monitor the installation**
	- The installation process is automated, but you can monitor progress using the console output of the nodes or by connecting to the rendezvous node via SSH (user: `core`).
	- Review logs on the rendezvous node for troubleshooting:
	  ```bash
	  journalctl -u agent
	  journalctl -u bootkube
	  ```

3. **Wait for completion**
	- The installer will configure the cluster, set up networking, and deploy all required OpenShift components.
	- Once complete, the kubeconfig file will be available in your working directory (e.g., `okd-install/auth/kubeconfig`).
  - Password for temporary admin user (```kubeadmin```) is in the file (`okd-install/auth/kubeadmin-password`)
  - **Installer logs:** The installer prints progress to the terminal. You can also tail the log file:
    ```sh
    tail -f okd-install/.openshift_install.log
    ```
  - **Bootstrap status:** Check if bootstrap is complete:
    ```sh
    openshift-install --dir okd-install wait-for bootstrap-complete --log-level=debug
    ```

4. **Access the cluster**
	- Use the `oc` CLI with the generated kubeconfig to verify cluster status:
	  ```bash
	  oc --kubeconfig okd-install/auth/kubeconfig get nodes
	  ```
	- Access the OpenShift web console. Address for the console is printed in the terminal when the install is completed.

### Troubleshooting

- If installation fails, check logs on the rendezvous node and verify network, DNS, NTP, and HAProxy configurations.
- Ensure all nodes are reachable and have correct static IPs and MAC addresses as defined in your configs.
- Redo ISO upload and boot process.

---

## 1. Summary Checklist

- [ ] Node hardware meets requirements
- [ ] VIPs and HAProxy configured
- [ ] DNS records created
- [ ] NTP working on all nodes
- [ ] `install-config.yaml` and `agent-config.yaml` created
- [ ] Backup taken before ISO creation
- [ ] Agent ISO created and ready for deployment

---
