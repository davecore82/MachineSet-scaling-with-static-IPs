# Validation: MachineSet scaling with static IPs from a reserved IP pool on vSphere (OpenShift 4.20)

| | |
|---|---|
| **Action item** | Test machine set scaling with static IPs from a reserved IP pool on vSphere |
| **Result** | ✅ **PASSED**: new nodes created by a MachineSet receive a static IP from a pool, wait for the IP before being created, and are cleaned up on scale-down |
| **OpenShift version** | 4.20.46 (installer-provisioned infrastructure, IPI) |
| **Platform** | VMware vSphere (lab environment), single network `VM-Network-01` (`192.168.100.0/24`) |
| **Date** | 2026-11-24 |

---

## 1. Summary

OpenShift 4.20 on vSphere can scale compute machines that use **static IP addresses taken from a reserved IP pool**, instead of DHCP. The flow is:

1. A `MachineSet` references an IP pool (`network.devices[].addressesFromPools`).
2. When the MachineSet scales up, the Machine API creates an **`IPAddressClaim`** for each new machine ("I need an IP from pool X") and **waits**. No VM is created yet.
3. An **IP address management (IPAM) controller** answers the claim by creating an **`IPAddress`** object (the actual IP, prefix and gateway) and binding it to the claim.
4. The Machine API then creates the VM and passes the static IP configuration to it. The node boots with that IP, with **no DHCP involved**.
5. On scale-down, the machine, VM, node and claim are removed automatically.

All five steps were confirmed in this test, with the output below.

> **Important for production:** OpenShift does **not** include an IPAM controller. Step 3 must be done by an IPAM controller you provide (typically integrated with your existing IPAM system). In this test, step 3 was done **manually** with `oc`, exactly as shown in the Red Hat documentation. See [section 6](#6-findings-and-production-considerations).

---

## 2. Test environment

| Item | Value |
|---|---|
| Cluster name | `ocp-test` (infrastructure ID `ocp-test-abcde`) |
| Topology | 3 control-plane nodes (compact cluster), plus 1 compute node added during the test |
| vSphere network | `VM-Network-01`, `192.168.100.0/24`, gateway `192.168.100.254` |
| DNS server | `192.168.99.10` |
| API VIP / Ingress VIP | `192.168.100.71` / `192.168.100.72` |
| Reserved static IPs | `192.168.100.65` bootstrap · `.66–.68` control plane · `.69–.70` pool for scaling |

---

## 3. How it works

```
 oc scale machineset ... --replicas=1
            │
            ▼
 ┌──────────────────────┐    creates     ┌─────────────────────────┐
 │ Machine API          │ ─────────────▶ │ IPAddressClaim          │
 │ (machine is waiting: │                │ poolRef: static-pool    │
 │  "WaitingForIPAddress│                └───────────┬─────────────┘
 │  no VM yet)          │                            │ answered by
 └──────────▲───────────┘                            ▼
            │ claim bound            ┌───────────────────────────────┐
            └─────────────────────── │ IPAM controller (customer)    │
                                     │ creates IPAddress 192.168.100.69 │
                                     │ and binds it to the claim     │
                                     └───────────────────────────────┘
            │
            ▼
 VM created with static IP 192.168.100.69/24, gw .254, DNS 192.168.99.10  →  node joins cluster
```

---

## 4. Test procedure and proof

### 4.1 Install the cluster with static IPs

Static IPs were configured in `install-config.yaml` with `platform.vsphere.hosts`. The documentation lists at least one statically addressed node as a prerequisite for machine scaling with static IPs.

Relevant part of `install-config.yaml` (credentials removed):

```yaml
platform:
  vsphere:
    apiVIPs:
    - 192.168.100.71
    ingressVIPs:
    - 192.168.100.72
    hosts:
    - role: bootstrap
      networkDevice:
        ipAddrs:
        - 192.168.100.65/24
        gateway: 192.168.100.254
        nameservers:
        - 192.168.99.10
    - role: control-plane
      networkDevice:
        ipAddrs:
        - 192.168.100.66/24
        gateway: 192.168.100.254
        nameservers:
        - 192.168.99.10
    - role: control-plane
      networkDevice:
        ipAddrs:
        - 192.168.100.67/24
        gateway: 192.168.100.254
        nameservers:
        - 192.168.99.10
    - role: control-plane
      networkDevice:
        ipAddrs:
        - 192.168.100.68/24
        gateway: 192.168.100.254
        nameservers:
        - 192.168.99.10
    failureDomains:
    - name: generated-failure-domain
      region: generated-region
      zone: generated-zone
      server: vcenter.example.com
      topology:
        datacenter: Datacenter
        computeCluster: /Datacenter/host/cluster01
        datastore: /Datacenter/datastore/vsanDatastore
        networks:
        - VM-Network-01
```

Install result:

```
$ openshift-install create cluster --dir cluster
...
level=info msg=Install complete!
level=info msg=Time elapsed: 36m42s
```

**Proof: cluster healthy, control-plane nodes have exactly the configured static IPs**

```
$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.20.46   True        False         3m45s   Cluster version is 4.20.46

$ oc get nodes -o wide
NAME                         STATUS  ROLES                        INTERNAL-IP
ocp-test-abcde-master-0  Ready   control-plane,master,worker  192.168.100.66
ocp-test-abcde-master-1  Ready   control-plane,master,worker  192.168.100.67
ocp-test-abcde-master-2  Ready   control-plane,master,worker  192.168.100.68
```

The IP claim types are present in the cluster, and the installer itself used the same claim mechanism for the control-plane static IPs (pool `default-0`):

```
$ oc get crd | grep ipam
ipaddressclaims.ipam.cluster.x-k8s.io    2026-11-24T16:13:52Z
ipaddresses.ipam.cluster.x-k8s.io        2026-11-24T16:13:50Z

$ oc get ipaddressclaims -n openshift-machine-api
NAME                                    POOL NAME   POOL KIND   AGE
ocp-test-abcde-master-0-claim-0-0   default-0   IPPool      29m
ocp-test-abcde-master-1-claim-0-0   default-0   IPPool      29m
ocp-test-abcde-master-2-claim-0-0   default-0   IPPool      30m
```

### 4.2 Create a MachineSet that takes its IPs from a pool

The MachineSet was copied from the installer-created worker MachineSet. Only the name, labels and network section were changed. The key part is `network.devices`:

```yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: ocp-test-abcde-static-pool
  namespace: openshift-machine-api
spec:
  replicas: 0
  # selector / template labels set to ocp-test-abcde-static-pool
  template:
    spec:
      providerSpec:
        value:
          apiVersion: machine.openshift.io/v1beta1
          kind: VSphereMachineProviderSpec
          credentialsSecret:
            name: vsphere-cloud-credentials
          diskGiB: 120
          memoryMiB: 16384
          numCPUs: 8
          numCoresPerSocket: 4
          network:
            devices:
            - addressesFromPools:            # <-- take the IP from a pool
              - group: ipamcontroller.example.io
                name: static-pool
                resource: IPPool
              nameservers:                   # <-- required for static IPs
              - 192.168.99.10
              networkName: VM-Network-01
          template: ocp-test-abcde-rhcos-generated-failure-domain
          userDataSecret:
            name: worker-user-data
          workspace:
            datacenter: Datacenter
            datastore: /Datacenter/datastore/vsanDatastore
            folder: /Datacenter/vm/ocp-test
            resourcePool: /Datacenter/host/cluster01/Resources
            server: vcenter.example.com
```

```
$ oc apply -f machineset-static-pool.json
machineset.machine.openshift.io/ocp-test-abcde-static-pool created
```

### 4.3 Scale up: a claim is created and the machine waits for an IP

```
$ oc scale machineset ocp-test-abcde-static-pool --replicas=1 -n openshift-machine-api
machineset.machine.openshift.io/ocp-test-abcde-static-pool scaled

$ oc get ipaddressclaims -n openshift-machine-api
NAME                                             POOL NAME     POOL KIND   AGE
ocp-test-abcde-master-0-claim-0-0            default-0     IPPool      30m
ocp-test-abcde-master-1-claim-0-0            default-0     IPPool      30m
ocp-test-abcde-master-2-claim-0-0            default-0     IPPool      30m
ocp-test-abcde-static-pool-4tnw2-claim-0-0   static-pool   IPPool      0s     <-- new claim

$ oc get machines -n openshift-machine-api | grep static-pool
ocp-test-abcde-static-pool-4tnw2   Provisioning
```

**Proof: the machine does not create a VM until it gets an IP**

```
$ oc get machine ocp-test-abcde-static-pool-4tnw2 -n openshift-machine-api \
    -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.reason} {.message}{"\n"}{end}'
Drainable=True
InstanceExists=False InstanceNotCreated Instance has not been created
Terminable=True

$ oc get machine ocp-test-abcde-static-pool-4tnw2 -n openshift-machine-api \
    -o jsonpath='{.status.providerStatus}'
{"conditions":[
  {"type":"IPAddressClaimed","status":"False","reason":"WaitingForIPAddress",
   "message":"Waiting on 1 IP address claims to be bound"},
  {"type":"MachineCreation","status":"False","reason":"MachineCreationSucceeded",
   "message":"error getting addresses from IP pool: error retrieving bound IP address:
              no IPAddress is bound to claim ocp-test-abcde-static-pool-4tnw2-claim-0-0"}]}
```

### 4.4 Answer the claim with a static IP (the IPAM controller's job, done manually here)

An `IPAddress` object was created with the next reserved IP, `192.168.100.69`:

```yaml
apiVersion: ipam.cluster.x-k8s.io/v1beta1
kind: IPAddress
metadata:
  name: ocp-test-abcde-static-pool-4tnw2-ipaddress-0-0
  namespace: openshift-machine-api
spec:
  address: 192.168.100.69
  prefix: 24
  gateway: 192.168.100.254
  claimRef:
    name: ocp-test-abcde-static-pool-4tnw2-claim-0-0
  poolRef:
    apiGroup: ipamcontroller.example.io
    kind: IPPool
    name: static-pool
```

```
$ oc create -f ipaddress-69.yaml
ipaddress.ipam.cluster.x-k8s.io/ocp-test-abcde-static-pool-4tnw2-ipaddress-0-0 created

$ oc patch ipaddressclaim ocp-test-abcde-static-pool-4tnw2-claim-0-0 -n openshift-machine-api \
    --type=merge --subresource=status \
    -p '{"status":{"addressRef":{"name":"ocp-test-abcde-static-pool-4tnw2-ipaddress-0-0"}}}'
ipaddressclaim.ipam.cluster.x-k8s.io/ocp-test-abcde-static-pool-4tnw2-claim-0-0 patched
```

**Proof: as soon as the claim is bound, the machine proceeds and the VM is created**

```
$ oc get machine ocp-test-abcde-static-pool-4tnw2 -n openshift-machine-api \
    -o jsonpath='{.status.phase}{"\n"}{range .status.providerStatus.conditions[*]}{.type}={.status} {.reason}: {.message}{"\n"}{end}'
Provisioning
IPAddressClaimed=True IPAddressesClaimed: All IP address claims are bound
MachineCreation=True MachineCreationSucceeded: Machine successfully created
```

### 4.5 The new node joins with the static IP

The node was **Ready about 2 minutes** after the claim was bound.

```
$ oc get machine ocp-test-abcde-static-pool-4tnw2 -n openshift-machine-api -o wide
NAME                                   PHASE     AGE     NODE                                   PROVIDERID                                       STATE
ocp-test-abcde-static-pool-4tnw2   Running   2m31s   ocp-test-abcde-static-pool-4tnw2   vsphere://422f3494-f3e5-81e6-c0fd-c69f030e55a4   poweredOn

$ oc get nodes -o wide
NAME                                  STATUS  ROLES                        INTERNAL-IP
ocp-test-abcde-master-0           Ready   control-plane,master,worker  192.168.100.66
ocp-test-abcde-master-1           Ready   control-plane,master,worker  192.168.100.67
ocp-test-abcde-master-2           Ready   control-plane,master,worker  192.168.100.68
ocp-test-abcde-static-pool-4tnw2  Ready   worker                       192.168.100.69   <-- static IP from the pool
```

**Proof: on the node itself, the IP is configured statically (not DHCP)**

```
$ oc debug node/ocp-test-abcde-static-pool-4tnw2 -- chroot /host \
    grep -E '^\[ipv4\]|^method|^address|^gateway|^dns' /etc/NetworkManager/system-connections/default_connection.nmconnection
[ipv4]
address1=192.168.100.69/24
dns=192.168.99.10;
gateway=192.168.100.254
method=manual

$ oc debug node/ocp-test-abcde-static-pool-4tnw2 -- chroot /host ip route show default
default via 192.168.100.254 dev br-ex proto static metric 48

$ oc debug node/ocp-test-abcde-static-pool-4tnw2 -- chroot /host grep nameserver /etc/resolv.conf
nameserver 192.168.100.69
nameserver 192.168.99.10

$ oc debug node/ocp-test-abcde-static-pool-4tnw2 -- chroot /host pgrep -a dhclient
(no DHCP client process)
```

> **Note:** the first `nameserver` is the node's own IP. That's expected on vSphere IPI: each node runs a local DNS service for cluster-internal names, then forwards to the configured DNS server (`192.168.99.10`).

### 4.6 Scale down: automatic cleanup

```
$ oc scale machineset ocp-test-abcde-static-pool --replicas=0 -n openshift-machine-api
machineset.machine.openshift.io/ocp-test-abcde-static-pool scaled
machine deleted after ~20s

$ oc get machines -n openshift-machine-api
NAME                          PHASE     AGE
ocp-test-abcde-master-0   Running   33m
ocp-test-abcde-master-1   Running   33m
ocp-test-abcde-master-2   Running   33m

$ oc get nodes
NAME                          STATUS   ROLES                         AGE   VERSION
ocp-test-abcde-master-0   Ready    control-plane,master,worker   33m   v1.33.13
ocp-test-abcde-master-1   Ready    control-plane,master,worker   33m   v1.33.13
ocp-test-abcde-master-2   Ready    control-plane,master,worker   33m   v1.33.13

$ oc get ipaddressclaims -n openshift-machine-api
NAME                                    POOL NAME   POOL KIND   AGE
ocp-test-abcde-master-0-claim-0-0   default-0   IPPool      33m
ocp-test-abcde-master-1-claim-0-0   default-0   IPPool      33m
ocp-test-abcde-master-2-claim-0-0   default-0   IPPool      33m      <-- static-pool claim removed

$ oc get ipaddresses.ipam.cluster.x-k8s.io -n openshift-machine-api
NAME                                                 ADDRESS       POOL NAME     POOL KIND
ocp-test-abcde-master-0-claim-0-0                192.168.100.66   default-0     IPPool
ocp-test-abcde-master-1-claim-0-0                192.168.100.67   default-0     IPPool
ocp-test-abcde-master-2-claim-0-0                192.168.100.68   default-0     IPPool
ocp-test-abcde-static-pool-4tnw2-ipaddress-0-0   192.168.100.69   static-pool   IPPool   <-- remains (see 6.3)
```

The machine, VM, node and `IPAddressClaim` were removed automatically. The manually created `IPAddress` stays until it's released (see [6.3](#63-the-ipam-controller-must-release-ips-on-scale-down)).

---

## 5. Results

| # | Check | Result |
|---|---|---|
| 1 | Install 4.20 IPI with static IPs | ✅ Control-plane nodes on `.66`, `.67`, `.68` |
| 2 | MachineSet with `addressesFromPools` accepted | ✅ |
| 3 | Scale-up creates an `IPAddressClaim` per machine | ✅ |
| 4 | Machine waits, with no VM created, until the claim is bound | ✅ `WaitingForIPAddress`, `InstanceExists=False` |
| 5 | Binding an `IPAddress` triggers VM creation | ✅ `All IP address claims are bound` → `Machine successfully created` |
| 6 | Node joins with the assigned static IP | ✅ `192.168.100.69`, Ready in about 2 minutes |
| 7 | IP is truly static on the node | ✅ `method=manual`, gateway and DNS as assigned, no DHCP client |
| 8 | Scale-down removes machine, VM, node and claim | ✅ About 20 seconds |

---

## 6. Findings and production considerations

### 6.1 You need an IPAM controller
OpenShift creates the IP requests (`IPAddressClaim`) but doesn't fulfil them. The 4.20 documentation states this is done by "your IP address management (IPAM) service". In production, deploy a controller that:
- watches `IPAddressClaim` objects in `openshift-machine-api` for your pool,
- allocates a free IP from your reserved range (ideally from your existing IPAM system),
- creates the matching `IPAddress` and binds it to the claim.

Without a controller, every scale-up waits until someone answers the claim by hand, as was done in this test.

### 6.2 Always set `nameservers` on pool-based MachineSets
When a cluster is installed with static IPs, the installer configures the default worker MachineSet to use a pool automatically (`installer.openshift.io` / `IPPool` / `default-0`), but **without `nameservers`**:

```yaml
network:
  devices:
  - addressesFromPools:
    - group: installer.openshift.io
      name: default-0
      resource: IPPool
    networkName: VM-Network-01
```

The documentation states that you must specify a name server for nodes that receive static IP addresses. Add `nameservers` to every MachineSet that uses `addressesFromPools`, as in section 4.2.

### 6.3 The IPAM controller must release IPs on scale-down
On scale-down, OpenShift deleted the machine and the `IPAddressClaim`, but **not** the `IPAddress`. Releasing the IP back to the pool is the IPAM controller's responsibility. Make sure your controller handles deletion, or IPs will leak.

### 6.4 Pool names are free-form
The `group`, `resource` and `name` in `addressesFromPools` only identify the pool for your IPAM controller. This test used the documentation's example values (`ipamcontroller.example.io`, `IPPool`, `static-pool`). Use whatever your controller expects.

### 6.5 Static IPs and the network
- Reserve the pool range in your IPAM or DHCP system so the same IPs are never handed out by DHCP.
- The vCenter account used by OpenShift needs `Network.Assign` on the port group.
- The installer host must trust the vCenter CA certificate (documented prerequisite). Without it, the installer fails its pre-flight check with `x509: certificate is not trusted`.

---

## 7. References

**Red Hat documentation (OpenShift 4.20)**
- [Machine scaling with static IP addresses](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/postinstallation_configuration/#nodes-vsphere-machine-set-scaling-static-ip_post-install-node-tasks): the procedure validated in this document
- [Installing on vSphere: installer-provisioned infrastructure](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_vmware_vsphere/installer-provisioned-infrastructure): see "Static IP addresses for vSphere nodes"
- [Creating a compute machine set on vSphere](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/machine_management/managing-compute-machines-with-the-machine-api)
- [Installation configuration parameters for vSphere](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_vmware_vsphere/installation-config-parameters-vsphere)

**Upstream API**
- `IPAddressClaim` / `IPAddress` (`ipam.cluster.x-k8s.io/v1beta1`) come from the Kubernetes Cluster API IPAM contract: <https://cluster-api.sigs.k8s.io/>

---

## Appendix A: Command quick reference

```bash
# Scale the pool-based MachineSet
oc scale machineset <machineset> --replicas=<n> -n openshift-machine-api

# Watch claims and IPs
oc get ipaddressclaims -n openshift-machine-api
oc get ipaddresses.ipam.cluster.x-k8s.io -n openshift-machine-api

# Why is a machine waiting?
oc get machine <machine> -n openshift-machine-api -o jsonpath='{.status.providerStatus}'

# Manually answer a claim (only without an IPAM controller)
oc create -f ipaddress.yaml
oc patch ipaddressclaim <claim> -n openshift-machine-api --type=merge --subresource=status \
  -p '{"status":{"addressRef":{"name":"<ipaddress-name>"}}}'

# Check node IP configuration
oc get nodes -o wide
oc debug node/<node> -- chroot /host cat /etc/NetworkManager/system-connections/default_connection.nmconnection
```
