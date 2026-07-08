# InfiniBand-Only Cluster Configuration — NVIDIA Base Command Manager (BCM 11)

**Primary source:** [Configuring NVIDIA Base Command Manager as an InfiniBand-Only Cluster](https://enterprise-support.nvidia.com/s/article/configuring-nvidia-base-command-manager-as-an-infiniband-only-cluster) (NVIDIA Enterprise Support)
**Cross-referenced against:** BCM 11 Administrator Manual §3.6 (Configuring InfiniBand Interfaces), §5.1.3 (iPXE Booting Using InfiniBand), §5.3.2–5.3.3 (Kernel Modules / InfiniBand Provisioning) — confirmed.

---

## 1. Required Kernel Modules

```
ib_core
ib_umad
ib_ipoib
ib_uverbs
mlx5_core
mlx5_ib
```

> ✅ **Confirmed on-site** with SLES 15 SP6 and ConnectX-7 NDR HCAs — this list is complete for that hardware/OS combination.
>
> Per Admin Manual §5.3.3, the exact module set is HCA- and distro-specific — there's no single universal list. BCM's own recommended method to determine it for other hardware/OS combos: stop and restart the InfiniBand service and diff the loaded modules:
> ```
> { service rdma stop; lsmod | cut -f1 -d" "; } > /tmp/a
> { service rdma start; lsmod | cut -f1 -d" "; } > /tmp/b
> diff /tmp/a /tmp/b
> ```
> **On SLES, substitute `openib` for the `rdma` service name** in the commands above (per §5.3.3, footnote-equivalent note). This matches the SLES 15 SP6 environment this list was confirmed on.
>
> For Red Hat 7+ where `rdma` can only be started (not stopped), use instead:
> ```
> { lsmod | cut -f1 -d" "; } > /tmp/a
> { systemctl start rdma-load-modules@rdma; lsmod | cut -f1 -d" "; } > /tmp/b
> ```

Modules are set per software image via `cmsh`:
```
% device use master
% softwareimage
% kernelmodules
% add <modulename>
% commit
```
*(Confirmed against BCM 11 Admin Manual §5.3.2.)*

> **Context (§3.6.1 vs §5.3.3):** for *regular post-boot use*, OFED/InfiniBand modules are auto-loaded by systemd at init — no manual kernelmodules config needed. The list above matters specifically for **provisioning over InfiniBand**, since provisioning happens pre-init, before the auto-load service runs — these modules have to be baked into the ramdisk explicitly.
>
> **Override hierarchy warning (§5.3.2):** kernel modules can be set at softwareimage, category, or device level. A lower level (device > category > image) **completely replaces** the module list from the level above — it does not merge with it. Setting modules at device/category level without including everything from the image level can silently break boot. Modules are normally best set at softwareimage level only.

---

## 2. Subnet Manager Configuration (if SM runs on the head node)

Every InfiniBand subnet needs at least one subnet manager running (§3.6.2). It's recommended to run 2 per subnet for redundancy. On a BCM cluster, the SM is best started via CMDaemon by assigning the role — first assign it, then configure:

```
% cmsh
% device roles headnode
% assign subnetmanager
% commit
```

> Note (§3.6.2 footnote): the subnet manager service is usually `opensm`, but is `opensmd` on SLES — this matches the `Restart opensmd daemon` prompt below and the SLES 15 SP6 environment this was tested on.

Then configure the role itself:

```
% cmsh
% device use master; roles;
[headnode->device[headnode]->roles[subnetmanager]]% show
Parameter                        Value
-------------------------------- ------------------------------------------------
Name                             subnetmanager
Revision
Type                             SubnetManagerRole
Add services                     yes
Interconnect                     IB
IB L2 MTU                        2K

[headnode->device[headnode]->roles[subnetmanager]]% set ibl2mtu 4K
[headnode->device*[headnode*]->roles*[subnetmanager*]]% commit
*** Restart opensmd daemon
```

- Default `IB L2 MTU` is `2K`; article recommends bumping to `4K`.
- Changing `ibl2mtu` requires restarting the `opensmd` daemon to take effect.
- Highly Recommend installing DOCA/MLNX-OFED before this step since on SLES the service is called opensm, its called opensmd when DOCA/MLNX-OFED is installed.

---

## 3. Key Elements / Gotchas

- **Build the cluster as Ethernet first** if IB cabling isn't in place yet. If you skip this and go IB-only with no physical IB link, you lose `cmsh` access entirely — no fallback path to fix it.
- Either:
  - Convert `internalnet` to InfiniBand, **or**
  - Add a new dedicated `ibnet` network
- If creating a new network, dell hpc recommended defaults (§3.6.3):

  | Property | Value |
  |---|---|
  | Name | `ibnet` |
  | Domain name | `ib.cluster` |
  | Type | `internal` |
  | Base address | `172.16.8.0` |
  | Netmask bits | `22` |
  | MTU | 4092 (datagram mode - recommended) /  up to 64k in connected mode |

  - Set the network as bootable and management-allowed:
    ```
    % cmsh
    % network use ibnet
    % set nodebooting yes
    % set managementallowed yes
    % commit
    ```
  - Set the category's management network to `ibnet` if it's going to be used that way.
- Set **CardType** to `Infiniband` on all relevant interfaces.
- Set **Connected Mode** to `no`. This matches the BCM default — InfiniBand interfaces default to **datagram mode** rather than connected mode, because datagram scales better (§3.6.3). If a node is PXE-booting or being provisioned over InfiniBand, the node-installer's mode setting must match:
  ```
  echo datagram > /cm/node-installer/scripts/ipoib_mode
  ```
- **BOOTIF caveat (§5.1.3 / §5.3.3):** the interface a node boots from (`BOOTIF`) must **not** already be configured as a separate interface for that node in CMDaemon. If `BOOTIF` is `ib0`, then `ib0` must not already exist as a configured interface — one or the other has to change. The manual recommends setting `BOOTIF` to `eth0` if an `ib0` device should also exist. *(This is exactly what the `remove bootif` step in the Post-Installation section below is resolving — see §4, step 5.)*
- **Bonding** (if used):
  - Only supported mode is `1` (active-backup).
  - Minimum `miimon=100` in options.
  - Example extended options: `primary=ib0 miimon=100 updelay=100 downdelay=100`
- Set the provisioning interface name accordingly.
- Set the management network at the partition level:
  ```
  % partition use base
  % set managementnetwork ibnet
  % commit
  ```
- Set IB MTU to `4K` (done via the `subnetmanager` role if the head node runs SM — see §2 above). Configure UFM if applicable.

---

## 4. Post-Installation: Converting to InfiniBand-Only

1. Configure `ibnet` to allow management and node booting:
   ```
   # cmsh
   % network use ibnet
   % set nodebooting yes
   % set managementallowed yes
   % commit
   ```
2. Set the base partition's management network to `ibnet`:
   ```
   % partition use base
   % set managementnetwork ibnet
   % commit
   ```
3. Set the default category's management network to `ibnet`:
   ```
   % category use default
   % set managementnetwork ibnet
   % commit
   ```
4. Set the head node's provisioning interface to `ib0`:
   ```
   % device use master
   % set provisioninginterface ib0
   % commit
   ```
5. Set the compute node's provisioning interface to `ib0`, and remove `bootif`:
   ```
   % device use node001
   % set provisioninginterface ib0
   % commit
   % interfaces
   % remove bootif
   % commit
   % quit
   ```
6. **Reboot the head node** — required to update management interfaces across config files.
7. After reboot, remove the now-unneeded Ethernet interface from the head node:
   ```
   # cmsh
   % device use master
   % interfaces
   % list
   Type      Network device name  IP              Network       Start if
   physical  eno1                 10.141.255.254  internalnet   always
   physical  ib0 [prov]           10.142.255.254  ibnet         always
   % remove eno1
   % commit
   ```
   *(interface name will vary by system — `eno1` used as example)*
8. Remove the `internalnet` network entirely:
   ```
   % partition use base
   % set externalnetwork ibnet
   % commit
   % network
   % use internalnet
   % set nodebooting no
   % set managementallowed no
   % commit
   % ..
   % remove internalnet
   % commit
   ```
9. **Reboot the head node again** to fully drop the Ethernet interface.
10. Cluster is now InfiniBand-only.

---
