# xCAT–GPU–Kubernetes OS Image Setup

## 1. Overview

This document describes how to prepare a Rocky Linux 9.6 xCAT netboot OS image for Kubernetes worker nodes with:

- Persistent storage for Kubernetes worker identity and state.
- Automatic worker-node bootstrap and cluster enrollment.
- systemd-managed bootstrap services.
- xCAT synclist distribution of worker bootstrap configuration and Kubernetes join material.
- Validation and troubleshooting procedures.

### Target OS image

```text
rocky9.6-x86_64-netboot-gpu-ai
```

### Image root

The examples assume:

```bash
export rootimgdir=/install/netboot/rocky9.6/x86_64/gpu-ai
```

> **Important:** Run image modification commands from the xCAT management/control node with appropriate privileges. Do not execute the image-root paths directly on a booted worker.

---

## 2. Architecture and Boot Flow

The intended worker boot sequence is:

```text
xCAT network boot
      |
      v
Rocky Linux 9.6 worker boots
      |
      +--> k8s-storage-bootstrap.service
      |        |
      |        +--> Detect / mount persistent worker storage
      |        +--> Restore persistent Kubernetes state
      |
      +--> k8s-worker-bootstrap.service
               |
               +--> Read worker-bootstrap.conf
               +--> Read join material from /run/xcat-k8s/
               +--> Verify prerequisites
               +--> Detect whether worker is already enrolled
               +--> kubeadm join (when required)
               +--> Start/enable kubelet
```

The key design principle is that **worker identity/state must reside on persistent storage**, not only in the ephemeral RAM-based root filesystem.

---

# 3. Prerequisites

Before modifying the OS image, verify:

- Rocky Linux 9.6 x86_64 xCAT netboot image exists.
- `rootimgdir` points to the intended image.
- The Kubernetes control plane is reachable from the worker network.
- `kubeadm`, `kubelet`, and the required container runtime are available in the OS image.
- Persistent storage is available to the worker and can be mounted during boot.
- The Kubernetes cluster is already initialized.
- The required Kubernetes bootstrap/join material has been generated according to your cluster's security model.
- The xCAT synclist used by the target OS image is known.

Verify the image:

```bash
export rootimgdir=/install/netboot/rocky9.6/x86_64/gpu-ai

test -d "$rootimgdir/rootimg" && echo "OS image root exists"
```

---

# 4. Prepare the Required Directories

Ensure the destination directories exist before copying files:

```bash
mkdir -p \
  "$rootimgdir/rootimg/usr/local/sbin" \
  "$rootimgdir/rootimg/etc/xcat/k8s" \
  "$rootimgdir/rootimg/etc/systemd/system"
```

Set ownership and permissions explicitly:

```bash
chown root:root \
  "$rootimgdir/rootimg/usr/local/sbin" \
  "$rootimgdir/rootimg/etc/xcat/k8s" \
  "$rootimgdir/rootimg/etc/systemd/system"
```

---

# 5. Install the Kubernetes Storage Bootstrap Script

Copy the persistent-storage bootstrap script into the OS image.

From the `k8s-worker` project directory:
```bash
copy the tar file from configruation_file path
cd ../configuration_file/
tar -xvf k8s-worker.tgz
cd k8s-worker
```


```bash
cp -a bootstrap_script/xcat-k8s-storage-bootstrap.sh \
  "$rootimgdir/rootimg/usr/local/sbin/"
```

Set secure ownership and permissions:

```bash
chmod 700 \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-storage-bootstrap.sh"

chown root:root \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-storage-bootstrap.sh"
```

Validate:

```bash
ls -l \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-storage-bootstrap.sh"
```

Expected characteristics:

```text
owner: root
group: root
mode: 700
```

---

# 6. Install the Kubernetes Worker Bootstrap Script

Copy the worker bootstrap script:

```bash
cp -a bootstrap_script/xcat-k8s-worker-bootstrap.sh \
  "$rootimgdir/rootimg/usr/local/sbin/"
```

Set secure ownership and permissions:

```bash
chmod 700 \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-worker-bootstrap.sh"

chown root:root \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-worker-bootstrap.sh"
```

Validate:

```bash
ls -l \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-worker-bootstrap.sh"
```

---

# 7. Install Worker Bootstrap Configuration

Copy the worker bootstrap configuration into the image:

```bash
install -D -m 600 -o root -g root \
  k8s_worker_conf/worker-bootstrap.conf \
  "$rootimgdir/rootimg/etc/xcat/k8s/worker-bootstrap.conf"
```

Validate:

```bash
ls -l \
  "$rootimgdir/rootimg/etc/xcat/k8s/worker-bootstrap.conf"
```

The configuration should not be world-readable because it may contain cluster bootstrap parameters or references to sensitive material.

Inspect only on a secured management node:

```bash
sed -n '1,200p' \
  "$rootimgdir/rootimg/etc/xcat/k8s/worker-bootstrap.conf"
```

> **Security note:** Do not place long-lived Kubernetes administrator credentials, client private keys, or unrestricted kubeconfig files in this configuration.

---

# 8. Install systemd Bootstrap Services

Copy both systemd units:

```bash
install -m 644 -o root -g root \
  systemd_service/k8s-storage-bootstrap.service \
  "$rootimgdir/rootimg/etc/systemd/system/k8s-storage-bootstrap.service"

install -m 644 -o root -g root \
  systemd_service/k8s-worker-bootstrap.service \
  "$rootimgdir/rootimg/etc/systemd/system/k8s-worker-bootstrap.service"
```

### Why `0644` for systemd units?

Systemd unit files normally do not need executable permissions. The executable bit belongs on the scripts referenced by `ExecStart=`.

Validate:

```bash
ls -l "$rootimgdir/rootimg/etc/systemd/system/" \
  k8s-storage-bootstrap.service \
  k8s-worker-bootstrap.service
```

---

# 9. Verify systemd Service Dependencies

The worker bootstrap service should run **after persistent storage is available**.

A recommended dependency model is:

```text
k8s-storage-bootstrap.service
            |
            v
k8s-worker-bootstrap.service
```

The worker service should not attempt to use Kubernetes state before the persistent-storage service has completed.

Example dependency section:

```ini
[Unit]
Description=Kubernetes Worker Bootstrap
Requires=k8s-storage-bootstrap.service
After=k8s-storage-bootstrap.service network-online.target
Wants=network-online.target
```

The exact dependency model should match the implementation of `xcat-k8s-storage-bootstrap.sh`.

After modifying the unit files in a running system, reload systemd:

```bash
systemctl daemon-reload
```

> For files placed directly into an offline xCAT image, `systemctl daemon-reload` on the xCAT management node does **not** reload the future worker's systemd state. The worker performs its own systemd initialization at boot. The unit files must also be enabled correctly in the image.

---

# 10. Enable the Bootstrap Services

For an offline OS image, prefer enabling the services through the image's systemd configuration rather than relying on a running host.

If the image is mounted/chrooted appropriately:

```bash
systemctl daemon-reload
systemctl enable k8s-storage-bootstrap.service
systemctl enable k8s-worker-bootstrap.service
```

If the image is not booted, verify that the unit enablement is actually represented in the image.

A common approach is:

```bash
ln -sf \
  /etc/systemd/system/k8s-storage-bootstrap.service \
  "$rootimgdir/rootimg/etc/systemd/system/multi-user.target.wants/k8s-storage-bootstrap.service"

ln -sf \
  /etc/systemd/system/k8s-worker-bootstrap.service \
  "$rootimgdir/rootimg/etc/systemd/system/multi-user.target.wants/k8s-worker-bootstrap.service"
```

Then validate:

```bash
ls -l \
  "$rootimgdir/rootimg/etc/systemd/system/multi-user.target.wants/" \
  | grep k8s-
```

> Use **one** enablement method appropriate to your xCAT image build workflow. Avoid creating duplicate or conflicting enablement links.

---

# 11. Kubernetes Worker Join Material

The worker requires bootstrap material to authenticate to the Kubernetes control plane.

For a standard kubeadm cluster, inspect available bootstrap tokens on the control plane:

```bash
kubeadm token list
```

A bootstrap token normally has a limited lifetime. Therefore, **do not treat a temporary kubeadm token as permanent worker identity**.

For a worker that must rejoin automatically after reboot, distinguish between:

1. **Worker identity/state**
   - kubelet configuration
   - kubelet client/server certificates as applicable
   - `/etc/kubernetes/`
   - container runtime state
   - node-specific persistent state

2. **Cluster bootstrap/join credentials**
   - kubeadm bootstrap token or equivalent
   - CA trust material
   - discovery information

The first category should be persisted with the worker. The second should be handled as a controlled bootstrap secret.

---

# 12. Bootstrap Token Distribution Through xCAT

Create the token file on the xCAT management node:

```bash
mkdir -p /xcatdata/token
chmod 700 /xcatdata/token
```

Create/update:

```bash
vim /xcatdata/token/bootstrap.token
```

Set restrictive permissions:

```bash
chmod 600 /xcatdata/token/bootstrap.token
chown root:root /xcatdata/token/bootstrap.token
```

Validate:

```bash
ls -l /xcatdata/token/bootstrap.token
```

> **Do not commit `bootstrap.token` to Git.** Treat it as a secret.

---

# 13. xCAT Synclist Configuration

Identify the synclist associated with the target OS image:

```bash
lsdef -t osimage rocky9.6-x86_64-netboot-gpu-ai
```

Extract the configured synclist:

```bash
lsdef -t osimage rocky9.6-x86_64-netboot-gpu-ai \
  | awk -F '=' '/synclists/ {print $2}'
```

Open the relevant synclist:

```bash
vim <path-to-synclist>
```

Add the required mappings:

```text
/xcatdata/worker-bootstrap.conf -> /etc/xcat/k8s/worker-bootstrap.conf
/xcatdata/token/bootstrap.token -> /run/xcat-k8s/bootstrap.token
```

Create the destination directory in the image or ensure the bootstrap process creates it:

```bash
mkdir -p "$rootimgdir/rootimg/run/xcat-k8s"
```

> `/run` is normally a runtime filesystem. Files copied there must be available at the correct point in the xCAT provisioning/boot process. Confirm the behavior of the specific xCAT synclist/provisioning mechanism used in your environment.

---

# 14. Validate the Image Before Deployment

Perform static validation before booting a worker.

### 14.1 Verify scripts

```bash
test -x \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-storage-bootstrap.sh"

test -x \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-worker-bootstrap.sh"
```

### 14.2 Verify configuration

```bash
test -f \
  "$rootimgdir/rootimg/etc/xcat/k8s/worker-bootstrap.conf"
```

### 14.3 Verify systemd units

```bash
test -f \
  "$rootimgdir/rootimg/etc/systemd/system/k8s-storage-bootstrap.service"

test -f \
  "$rootimgdir/rootimg/etc/systemd/system/k8s-worker-bootstrap.service"
```

### 14.4 Verify permissions

```bash
stat -c '%U:%G %a %n' \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-storage-bootstrap.sh" \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-worker-bootstrap.sh" \
  "$rootimgdir/rootimg/etc/xcat/k8s/worker-bootstrap.conf"
```

Expected:

```text
root:root 700 ...xcat-k8s-storage-bootstrap.sh
root:root 700 ...xcat-k8s-worker-bootstrap.sh
root:root 600 ...worker-bootstrap.conf
```

---

# 15. Validate systemd Unit Configuration

If the image can be safely inspected using a chroot:

```bash
chroot "$rootimgdir/rootimg" systemd-analyze verify \
  /etc/systemd/system/k8s-storage-bootstrap.service

chroot "$rootimgdir/rootimg" systemd-analyze verify \
  /etc/systemd/system/k8s-worker-bootstrap.service
```

If the image cannot be chrooted because required runtime dependencies are unavailable, perform the verification on an equivalent Rocky Linux environment and inspect the unit files manually.

---

# 16. Worker-Side Validation

After provisioning a worker, verify the services:

```bash
systemctl status k8s-storage-bootstrap.service
systemctl status k8s-worker-bootstrap.service
```

Check whether they are enabled:

```bash
systemctl is-enabled k8s-storage-bootstrap.service
systemctl is-enabled k8s-worker-bootstrap.service
```

Expected:

```text
enabled
enabled
```

Check execution results:

```bash
systemctl is-active k8s-storage-bootstrap.service
systemctl is-active k8s-worker-bootstrap.service
```

---

# 17. Check Bootstrap Logs

Use journald:

```bash
journalctl -u k8s-storage-bootstrap.service -b --no-pager
```

```bash
journalctl -u k8s-worker-bootstrap.service -b --no-pager
```

For live debugging:

```bash
journalctl -fu k8s-worker-bootstrap.service
```

Check the previous boot if troubleshooting reboot behavior:

```bash
journalctl -u k8s-worker-bootstrap.service -b -1 --no-pager
```

---

# 18. Verify Persistent Storage

Confirm the persistent device and mount:

```bash
lsblk -f
```

```bash
findmnt
```

If the implementation uses a dedicated mount point:

```bash
findmnt <persistent-mount-point>
```

Verify that the expected Kubernetes state exists on persistent storage.

For example:

```bash
ls -la /etc/kubernetes/
```

and, where applicable:

```bash
ls -la <persistent-kubernetes-state-path>
```

The exact path must match the storage-bootstrap implementation.

---

# 19. Verify Kubernetes Worker Registration

From the control plane:

```bash
kubectl get nodes -o wide
```

Inspect the worker:

```bash
kubectl describe node <worker-node-name>
```

Verify kubelet:

```bash
systemctl status kubelet
```

Check kubelet logs:

```bash
journalctl -u kubelet -b --no-pager
```

A successfully bootstrapped worker should eventually report:

```text
Ready
```

in:

```bash
kubectl get nodes
```

---

# 20. Reboot Persistence Test

This is the most important validation for the persistent-worker design.

## Test 1 — Initial boot

Record:

```bash
hostnamectl
```

```bash
kubectl get node <worker-node-name> -o yaml
```

Record at minimum:

- Kubernetes node name.
- Node UID.
- Worker hostname.
- Internal IP.
- Kubelet identity.
- Relevant `/etc/kubernetes` state.
- Persistent storage mount.
- Service execution status.

## Test 2 — Reboot

On the worker:

```bash
reboot
```

After reboot:

```bash
hostnamectl
```

```bash
systemctl status k8s-storage-bootstrap.service
systemctl status k8s-worker-bootstrap.service
systemctl status kubelet
```

From the control plane:

```bash
kubectl get nodes -o wide
```

Then compare the node object:

```bash
kubectl get node <worker-node-name> -o yaml
```

## Expected behavior

If the worker's Kubernetes identity is correctly persisted:

- Persistent Kubernetes state remains available.
- The kubelet can recover its identity.
- The worker reconnects to the existing Kubernetes node identity.
- The control plane should not unnecessarily receive a brand-new worker identity.

The exact node UID behavior should be measured from the testbed rather than assumed.

---

# 21. Hostname Change Test

To investigate the distinction between operating-system hostname and Kubernetes node identity:

Before changing the hostname, record:

```bash
hostnamectl
```

```bash
kubectl get node <worker-node-name> -o yaml
```

Change the hostname:

```bash
hostnamectl set-hostname <new-hostname>
```

Reboot:

```bash
reboot
```

After reboot, collect:

```bash
hostnamectl
```

```bash
cat /etc/hostname
```

```bash
systemctl status kubelet
```

From the control plane:

```bash
kubectl get nodes -o wide
```

```bash
kubectl get nodes -o yaml
```

Compare:

- Linux hostname.
- Kubernetes node name.
- Kubernetes node UID.
- Kubelet credentials.
- Node object recreation/update.
- Persistent Kubernetes state.

> **Do not conclude that hostname equality alone determines Kubernetes node identity.** Kubernetes node identity involves the Node API object and kubelet-side state/credentials. The test should establish the actual behavior of the deployed versions and configuration.

---

# 22. Security Best Practices

## 22.1 Protect bootstrap credentials

Use:

```bash
chmod 600 /xcatdata/token/bootstrap.token
chown root:root /xcatdata/token/bootstrap.token
```

Avoid:

```text
chmod 644 bootstrap.token
```

Do not store tokens in:

- Git repositories.
- Public artifact repositories.
- Container images.
- World-readable xCAT directories.

## 22.2 Prefer short-lived bootstrap credentials

A kubeadm bootstrap token is intended for cluster bootstrapping and normally has a limited lifetime.

For long-lived workers, rely on persisted node identity/credentials where appropriate rather than continuously distributing a permanent bootstrap token.

## 22.3 Restrict configuration permissions

Use:

```text
worker-bootstrap.conf       0600
bootstrap.token             0600
bootstrap scripts            0700
systemd unit files           0644
```

## 22.4 Avoid logging secrets

Bootstrap scripts should never log:

- Full kubeadm tokens.
- Private keys.
- Client certificates containing sensitive material.
- kubeconfig credentials.

When debugging, redact secrets before collecting logs.

---

# 23. Recommended Bootstrap Script Behavior

The worker bootstrap script should be **idempotent**.

A recommended decision flow is:

```text
Start
  |
  v
Verify required commands
  |
  v
Verify network connectivity
  |
  v
Verify persistent storage
  |
  v
Check existing Kubernetes identity/state
  |
  +---- Existing state ----> Start/reconcile kubelet
  |
  +---- No state ----------> Validate bootstrap material
                                |
                                v
                            kubeadm join
                                |
                                v
                          Persist state
                                |
                                v
                           Start kubelet
```

The script should not blindly execute:

```bash
kubeadm join ...
```

on every boot.

Repeated joins can create operational problems and should be avoided when the node already has valid persistent Kubernetes state.

---

# 24. Operational Troubleshooting

## Storage bootstrap failed

Check:

```bash
systemctl status k8s-storage-bootstrap.service
journalctl -u k8s-storage-bootstrap.service -b --no-pager
lsblk -f
findmnt
```

Verify:

- Correct block device.
- Correct filesystem.
- Correct UUID.
- Correct mount point.
- Persistent storage is available early enough during boot.

---

## Worker bootstrap failed

Check:

```bash
systemctl status k8s-worker-bootstrap.service
journalctl -u k8s-worker-bootstrap.service -b --no-pager
```

Then verify:

```bash
ls -l /etc/xcat/k8s/worker-bootstrap.conf
ls -l /run/xcat-k8s/
```

Verify network access to the Kubernetes API server:

```bash
curl -k https://<kubernetes-api-server>:6443/healthz
```

Use the appropriate API endpoint and security controls for your environment.

---

## Worker is not `Ready`

Check:

```bash
kubectl get nodes -o wide
```

```bash
kubectl describe node <worker-node-name>
```

On the worker:

```bash
systemctl status kubelet
journalctl -u kubelet -b --no-pager
```

Also verify the container runtime:

```bash
systemctl status containerd
```

---

## Worker joins as a new node after reboot

Investigate whether persistent Kubernetes state survived:

```bash
findmnt <persistent-mount-point>
```

```bash
ls -la <persistent-kubernetes-state-path>
```

Verify that the storage bootstrap service completes **before** the worker bootstrap service.

Also compare:

```bash
kubectl get node <old-node-name> -o yaml
kubectl get nodes -o wide
```

The important question is whether the worker recovered the same kubelet-side identity/state or performed a fresh `kubeadm join`.

---

# 25. Final Pre-Deployment Checklist

Before deploying the image to production or a larger GPU worker pool:

- [ ] Correct Rocky Linux 9.6 OS image selected.
- [ ] `rootimgdir` verified.
- [ ] Storage bootstrap script installed.
- [ ] Worker bootstrap script installed.
- [ ] Scripts owned by `root:root`.
- [ ] Scripts have mode `0700`.
- [ ] `worker-bootstrap.conf` installed.
- [ ] Configuration has mode `0600`.
- [ ] systemd units installed.
- [ ] systemd units have mode `0644`.
- [ ] Storage service runs before worker service.
- [ ] Both services are enabled.
- [ ] xCAT synclist contains worker configuration mapping.
- [ ] Bootstrap token mapping is present where required.
- [ ] Bootstrap token is protected and excluded from source control.
- [ ] API-server connectivity verified.
- [ ] Container runtime enabled.
- [ ] kubelet enabled.
- [ ] First-boot worker join tested.
- [ ] Reboot persistence tested.
- [ ] Hostname-change test completed.
- [ ] Kubernetes Node UID and node object behavior recorded.
- [ ] Logs verified after reboot.
- [ ] No bootstrap secrets appear in logs.

---

# 26. Quick Installation Reference

For a repeatable image build, the core installation sequence is:

```bash
export rootimgdir=/install/netboot/rocky9.6/x86_64/gpu-ai

mkdir -p \
  "$rootimgdir/rootimg/usr/local/sbin" \
  "$rootimgdir/rootimg/etc/xcat/k8s" \
  "$rootimgdir/rootimg/etc/systemd/system"

# Kubernetes bootstrap scripts
install -m 700 -o root -g root \
  bootstrap_script/xcat-k8s-storage-bootstrap.sh \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-storage-bootstrap.sh"

install -m 700 -o root -g root \
  bootstrap_script/xcat-k8s-worker-bootstrap.sh \
  "$rootimgdir/rootimg/usr/local/sbin/xcat-k8s-worker-bootstrap.sh"

# Worker configuration
install -D -m 600 -o root -g root \
  k8s_worker_conf/worker-bootstrap.conf \
  "$rootimgdir/rootimg/etc/xcat/k8s/worker-bootstrap.conf"

# systemd services
install -m 644 -o root -g root \
  systemd_service/k8s-storage-bootstrap.service \
  "$rootimgdir/rootimg/etc/systemd/system/k8s-storage-bootstrap.service"

install -m 644 -o root -g root \
  systemd_service/k8s-worker-bootstrap.service \
  "$rootimgdir/rootimg/etc/systemd/system/k8s-worker-bootstrap.service"
```

Then configure service enablement and the xCAT synclist according to the image build/provisioning method.

---

# 27. Important Design Notes

### Persistent storage is the source of worker identity

For a stateless xCAT worker, the RAM-based root filesystem is disposable. Any Kubernetes state stored only there can disappear after reboot.

Therefore, the persistence mechanism must cover the state required by kubelet/container runtime and the worker bootstrap logic.

### Bootstrap credentials are not the same as node identity

A bootstrap token allows a node to initially authenticate and join the cluster. It should not be confused with the worker's long-term Kubernetes identity.

### xCAT provisioning and Kubernetes enrollment are separate layers

The architecture has three distinct responsibilities:

```text
xCAT
 |
 +-- Provides OS image
 +-- Distributes configuration/secrets
 |
 v
Worker OS
 |
 +-- Restores persistent state
 +-- Runs bootstrap services
 |
 v
Kubernetes
 |
 +-- Authenticates kubelet
 +-- Maintains Node object
 +-- Schedules workloads
```

Keeping these layers separate makes troubleshooting and future automation significantly easier.

---

# 28. Change Management

For production changes, maintain version-controlled copies of:

```text
bootstrap_script/
├── xcat-k8s-storage-bootstrap.sh
└── xcat-k8s-worker-bootstrap.sh

k8s_worker_conf/
└── worker-bootstrap.conf

systemd_service/
├── k8s-storage-bootstrap.service
└── k8s-worker-bootstrap.service
```

Do **not** version-control:

```text
/xcatdata/token/bootstrap.token
```

Instead, document how the secret is generated and deployed.

Recommended workflow:

```text
Change
  |
  v
Static validation
  |
  v
Build/update OS image
  |
  v
Deploy to one test worker
  |
  v
Validate first boot
  |
  v
Reboot test
  |
  v
Hostname/identity test
  |
  v
Control-plane validation
  |
  v
Production rollout
```

This reduces the risk of propagating a bootstrap or identity problem to an entire GPU worker pool.
