# Production PoC: xCAT-Managed Kubernetes Worker Bootstrap
## Recommended Implementation — Points 40–50

**Purpose:** Production-oriented PoC runbook for automatically enrolling and managing Kubernetes worker nodes provisioned by xCAT, while preserving Kubernetes worker identity across reboots.

**Scope:** Fresh worker bootstrap, persistent local Kubernetes state, short-lived bootstrap credentials, idempotent reboot behavior, controlled recovery/re-registration, xCAT synchronization, and validation.

---

# 1. Target Production Architecture

```text
                    Kubernetes Control Plane
                    ------------------------
                    kube-apiserver :6443
                           |
                    kubeadm bootstrap
                           |
                 Short-lived bootstrap token
                           |
                           v
+------------------------------------------------------+
|                 xCAT Management Node                 |
|                                                      |
|  OS/Image Provisioning                               |
|  xCAT Synclist                                       |
|  Per-node bootstrap orchestration                   |
|  Token creation request                              |
+---------------------------+--------------------------+
                            |
                            | xCAT / approved SSH
                            | token transfer
                            v
+------------------------------------------------------+
|                 Kubernetes Worker                    |
|                                                      |
|  Stateless OS image                                  |
|       |                                              |
|       +--> /local/k8s  <-- persistent local NVMe    |
|              |                                       |
|              +-- etc-kubernetes -> /etc/kubernetes  |
|              +-- kubelet        -> /var/lib/kubelet |
|              +-- containerd     -> /var/lib/containerd
|                                                      |
|  xcat-k8s-storage-bootstrap.service                 |
|                  |                                   |
|                  v                                   |
|  xcat-k8s-worker-bootstrap.service                  |
|                  |                                   |
|          existing identity?                         |
|            /          \                              |
|          YES           NO                           |
|           |             |                            |
|       start kubelet   short-lived token             |
|                         |                            |
|                    kubeadm join                     |
|                         |                            |
|                    kubelet identity                 |
+------------------------------------------------------+
```

## Required behavior

| Event | Required behavior |
|---|---|
| First boot | Prepare persistent storage and perform one `kubeadm join` |
| Normal reboot | Never run `kubeadm join` again |
| Existing valid identity | Start kubelet and use existing identity |
| Missing identity | Do not silently generate credentials |
| Missing/expired token | Enter controlled bootstrap/recovery state |
| Join failure | Delete bootstrap token and report failure |
| Worker replacement | Explicitly remove old Kubernetes identity and perform fresh registration |
| xCAT resync | Must not overwrite Kubernetes identity |
| Kubelet certificate renewal | Use kubelet certificate rotation, not repeated `kubeadm join` |

---

# 2. PoC Assumptions

Replace these placeholders before deployment:

```text
<K8S_API_ENDPOINT>       Kubernetes API endpoint / VIP
<CONTROL_PLANE_HOST>     Control-plane host used to create bootstrap tokens
<K8S_CA_CERT_HASH>       SHA256 public-key hash of cluster CA
<WORKER_NODE>             xCAT/Kubernetes worker name
```

Example:

```text
K8S_API_ENDPOINT=10.10.10.100:6443
CONTROL_PLANE_HOST=10.10.10.101
WORKER_NODE=hopper01
```

The worker must already have:

- xCAT-provisioned OS
- Kubernetes packages
- `kubeadm`
- `kubelet`
- `containerd`
- working network connectivity to the Kubernetes API
- correctly configured containerd CRI
- local NVMe/SSD partition available for Kubernetes persistent state

---

# 3. Persistent Worker Storage

The worker uses local SSD/NVMe for Kubernetes state.

Required persistent paths:

```text
/local/k8s/
├── etc-kubernetes/
├── kubelet/
├── containerd/
└── bootstrap/
```

Bind mounts:

```text
/local/k8s/etc-kubernetes -> /etc/kubernetes
/local/k8s/kubelet        -> /var/lib/kubelet
/local/k8s/containerd     -> /var/lib/containerd
```

The storage bootstrap script must **never automatically format a disk**.

## 3.1 Storage bootstrap script

Create:

```text
/usr/local/sbin/xcat-k8s-storage-bootstrap.sh
```

```bash
#!/bin/bash
set -euo pipefail

DEVICE="${K8S_STORAGE_DEVICE:-/dev/nvme0n1p3}"
MOUNTPOINT="/local/k8s"
EXPECTED_FSTYPE="xfs"

log() {
    logger -t xcat-k8s-storage-bootstrap -- "$*"
    echo "$(date -Is) $*"
}

fail() {
    log "ERROR: $*"
    exit 1
}

[[ $EUID -eq 0 ]] || fail "Must run as root"

mkdir -p "$MOUNTPOINT"

if ! blkid "$DEVICE" >/dev/null 2>&1; then
    fail "Device $DEVICE has no filesystem. Refusing to format automatically."
fi

FSTYPE="$(blkid -o value -s TYPE "$DEVICE")"

[[ "$FSTYPE" == "$EXPECTED_FSTYPE" ]] || \
    fail "Unexpected filesystem type on $DEVICE: $FSTYPE"

if ! findmnt -rn -S "$DEVICE" -T "$MOUNTPOINT" >/dev/null 2>&1; then
    mount "$DEVICE" "$MOUNTPOINT"
fi

findmnt -rn -T "$MOUNTPOINT" >/dev/null || \
    fail "$MOUNTPOINT is not mounted"

mkdir -p \
    "$MOUNTPOINT/etc-kubernetes" \
    "$MOUNTPOINT/kubelet" \
    "$MOUNTPOINT/containerd" \
    "$MOUNTPOINT/bootstrap"

mountpoint -q /etc/kubernetes || {
    mkdir -p /etc/kubernetes
    mount --bind "$MOUNTPOINT/etc-kubernetes" /etc/kubernetes
}

mountpoint -q /var/lib/kubelet || {
    mkdir -p /var/lib/kubelet
    mount --bind "$MOUNTPOINT/kubelet" /var/lib/kubelet
}

mountpoint -q /var/lib/containerd || {
    mkdir -p /var/lib/containerd
    mount --bind "$MOUNTPOINT/containerd" /var/lib/containerd
}

log "Persistent Kubernetes storage is ready"
```

Set permissions:

```bash
chmod 0750 /usr/local/sbin/xcat-k8s-storage-bootstrap.sh
chown root:root /usr/local/sbin/xcat-k8s-storage-bootstrap.sh
```

---

# 4. Kubernetes Worker Configuration

Create:

```text
/etc/xcat/k8s/worker-bootstrap.conf
```

```ini
K8S_API_ENDPOINT="<K8S_API_ENDPOINT>:6443"
K8S_CA_CERT_HASH="sha256:<K8S_CA_CERT_HASH>"
K8S_CRI_SOCKET="unix:///run/containerd/containerd.sock"

K8S_STORAGE_ROOT="/local/k8s"
K8S_CONFIG="/etc/kubernetes"
KUBELET_DIR="/var/lib/kubelet"
CONTAINERD_DIR="/var/lib/containerd"

BOOTSTRAP_TOKEN_TTL="15m"
BOOTSTRAP_TOKEN_DIR="/run/xcat-k8s"

NODE_STATE_FILE="/local/k8s/bootstrap/state"
```

Do **not** place a bootstrap token in this configuration file.

Do **not** place:

```text
admin.conf
kubelet.conf
kubelet private keys
long-lived bootstrap tokens
```

in the xCAT image.

---

# 5. Obtain the Kubernetes CA Pin

Run on the Kubernetes control plane:

```bash
cat /etc/kubernetes/pki/ca.crt \
  | openssl x509 -pubkey \
  | openssl rsa -pubin -outform der 2>/dev/null \
  | openssl dgst -sha256 -hex
```

Example result:

```text
SHA2-256(stdin)= 0123456789abcdef...
```

Configure:

```ini
K8S_CA_CERT_HASH="sha256:0123456789abcdef..."
```

The resulting value must contain exactly the SHA256 hash of the CA public key.

The worker join must use:

```bash
--discovery-token-ca-cert-hash "$K8S_CA_CERT_HASH"
```

Never use:

```bash
--discovery-token-unsafe-skip-ca-verification
```

---

# 6. systemd Storage Service

Create:

```text
/etc/systemd/system/xcat-k8s-storage-bootstrap.service
```

```ini
[Unit]
Description=xCAT Kubernetes local persistent storage bootstrap
After=local-fs.target systemd-udev-settle.service
Before=containerd.service kubelet.service
Wants=systemd-udev-settle.service
RequiresMountsFor=/local/k8s

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/xcat-k8s-storage-bootstrap.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

---

# 7. Worker Bootstrap Service

Create:

```text
/etc/systemd/system/xcat-k8s-worker-bootstrap.service
```

```ini
[Unit]
Description=xCAT Kubernetes worker bootstrap
Requires=xcat-k8s-storage-bootstrap.service
After=xcat-k8s-storage-bootstrap.service containerd.service
Before=kubelet.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/xcat-k8s-worker-bootstrap.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

---

# 8. systemd Ordering for containerd

Create:

```text
/etc/systemd/system/containerd.service.d/10-xcat-storage.conf
```

```ini
[Unit]
Requires=xcat-k8s-storage-bootstrap.service
After=xcat-k8s-storage-bootstrap.service
```

---

# 9. systemd Ordering for kubelet

Create:

```text
/etc/systemd/system/kubelet.service.d/10-xcat-bootstrap.conf
```

```ini
[Unit]
Requires=xcat-k8s-storage-bootstrap.service
After=xcat-k8s-storage-bootstrap.service
```

Reload systemd:

```bash
systemctl daemon-reload
```

Enable the storage/bootstrap services:

```bash
systemctl enable xcat-k8s-storage-bootstrap.service
systemctl enable xcat-k8s-worker-bootstrap.service
```

Enable normal services:

```bash
systemctl enable containerd
systemctl enable kubelet
```

---

# 10. Worker Bootstrap Script

Create:

```text
/usr/local/sbin/xcat-k8s-worker-bootstrap.sh
```

```bash
#!/bin/bash
set -euo pipefail

CONFIG="/etc/xcat/k8s/worker-bootstrap.conf"

[[ -r "$CONFIG" ]] || {
    logger -t xcat-k8s-worker-bootstrap "ERROR: Missing $CONFIG"
    exit 1
}

source "$CONFIG"

log() {
    logger -t xcat-k8s-worker-bootstrap -- "$*"
    echo "$(date -Is) $*"
}

set_state() {
    mkdir -p "$(dirname "$NODE_STATE_FILE")"
    printf '%s\n' "$1" > "$NODE_STATE_FILE"
    chmod 0600 "$NODE_STATE_FILE"
    log "State=$1"
}

fail() {
    set_state "BOOTSTRAP_FAILED"
    log "ERROR: $*"
    exit 1
}

[[ $EUID -eq 0 ]] || fail "Must run as root"

findmnt -rn -T "$K8S_STORAGE_ROOT" >/dev/null || \
    fail "$K8S_STORAGE_ROOT is not mounted"

mountpoint -q "$K8S_CONFIG" || fail "$K8S_CONFIG is not a mountpoint"
mountpoint -q "$KUBELET_DIR" || fail "$KUBELET_DIR is not a mountpoint"
mountpoint -q "$CONTAINERD_DIR" || fail "$CONTAINERD_DIR is not a mountpoint"

mkdir -p "$BOOTSTRAP_TOKEN_DIR"
chmod 0700 "$BOOTSTRAP_TOKEN_DIR"

TOKEN_FILE="$BOOTSTRAP_TOKEN_DIR/bootstrap.token"

systemctl start containerd
systemctl is-active --quiet containerd || \
    fail "containerd is not active"

# ---------------------------------------------------------
# Existing Kubernetes identity
# ---------------------------------------------------------

if [[ -s "$K8S_CONFIG/kubelet.conf" ]] && \
   [[ -s "$KUBELET_DIR/pki/kubelet-client-current.pem" ]]; then

    if openssl x509 \
        -in "$KUBELET_DIR/pki/kubelet-client-current.pem" \
        -checkend 0 \
        -noout >/dev/null 2>&1; then

        if openssl x509 \
            -in "$KUBELET_DIR/pki/kubelet-client-current.pem" \
            -noout -subject 2>/dev/null \
            | grep -q 'system:node:'; then

            set_state "JOINED"
            systemctl enable kubelet >/dev/null 2>&1 || true
            systemctl restart kubelet
            log "Existing valid kubelet identity found; kubeadm join is NOT required"
            exit 0
        fi
    fi
fi

# ---------------------------------------------------------
# No valid identity: controlled bootstrap is required
# ---------------------------------------------------------

if [[ ! -s "$TOKEN_FILE" ]]; then
    set_state "BOOTSTRAP_REQUIRED"
    log "No valid kubelet identity and no bootstrap token"
    exit 1
fi

chmod 0600 "$TOKEN_FILE"
chown root:root "$TOKEN_FILE"

TOKEN="$(cat "$TOKEN_FILE")"

if ! [[ "$TOKEN" =~ ^[a-z0-9]{6}\.[a-z0-9]{16}$ ]]; then
    rm -f "$TOKEN_FILE"
    fail "Invalid bootstrap token format"
fi

set_state "BOOTSTRAPPING"

JOIN_RC=0

if kubeadm join "$K8S_API_ENDPOINT" \
    --token "$TOKEN" \
    --discovery-token-ca-cert-hash "$K8S_CA_CERT_HASH" \
    --cri-socket "$K8S_CRI_SOCKET" \
    --node-name "$(hostname -s)"
then
    JOIN_RC=0
else
    JOIN_RC=$?
fi

# Bootstrap credential is always destroyed after use.
rm -f "$TOKEN_FILE"

if [[ "$JOIN_RC" -ne 0 ]]; then
    fail "kubeadm join failed with rc=$JOIN_RC"
fi

# ---------------------------------------------------------
# Verify resulting identity
# ---------------------------------------------------------

[[ -s "$K8S_CONFIG/kubelet.conf" ]] || \
    fail "kubeadm join completed but kubelet.conf is missing"

[[ -s "$KUBELET_DIR/pki/kubelet-client-current.pem" ]] || \
    fail "kubeadm join completed but kubelet client certificate is missing"

set_state "JOINED"

systemctl enable kubelet
systemctl restart kubelet

systemctl is-active --quiet kubelet || \
    fail "kubelet failed after successful join"

log "Worker bootstrap completed successfully"
exit 0
```

Set permissions:

```bash
chmod 0750 /usr/local/sbin/xcat-k8s-worker-bootstrap.sh
chown root:root /usr/local/sbin/xcat-k8s-worker-bootstrap.sh
```

---

# 11. Enable Services

On the worker:

```bash
systemctl daemon-reload

systemctl enable xcat-k8s-storage-bootstrap.service
systemctl enable xcat-k8s-worker-bootstrap.service
systemctl enable containerd
systemctl enable kubelet
```

Verify:

```bash
systemctl is-enabled xcat-k8s-storage-bootstrap.service
systemctl is-enabled xcat-k8s-worker-bootstrap.service
systemctl is-enabled containerd
systemctl is-enabled kubelet
```

---

# 12. xCAT Synclist

On the xCAT management node create:

```text
/opt/xcat/k8s/synclists/k8s-worker.synclist
```

```text
/opt/xcat/k8s/config/worker-bootstrap.conf -> /etc/xcat/k8s/worker-bootstrap.conf
/opt/xcat/k8s/scripts/xcat-k8s-worker-bootstrap.sh -> /usr/local/sbin/xcat-k8s-worker-bootstrap.sh
/opt/xcat/k8s/scripts/xcat-k8s-storage-bootstrap.sh -> /usr/local/sbin/xcat-k8s-storage-bootstrap.sh
/opt/xcat/k8s/systemd/xcat-k8s-storage-bootstrap.service -> /etc/systemd/system/xcat-k8s-storage-bootstrap.service
/opt/xcat/k8s/systemd/xcat-k8s-worker-bootstrap.service -> /etc/systemd/system/xcat-k8s-worker-bootstrap.service
/opt/xcat/k8s/systemd/kubelet.service.d/10-xcat-bootstrap.conf -> /etc/systemd/system/kubelet.service.d/10-xcat-bootstrap.conf
/opt/xcat/k8s/systemd/containerd.service.d/10-xcat-storage.conf -> /etc/systemd/system/containerd.service.d/10-xcat-storage.conf
```

Create source directories:

```bash
mkdir -p \
    /opt/xcat/k8s/config \
    /opt/xcat/k8s/scripts \
    /opt/xcat/k8s/systemd/kubelet.service.d \
    /opt/xcat/k8s/systemd/containerd.service.d \
    /opt/xcat/k8s/synclists
```

Copy the files into their corresponding source locations.

---

# 13. Synchronize xCAT Configuration

For a PoC worker:

```bash
updatenode hopper01 -F
```

Or use the xCAT file synchronization mechanism appropriate to the existing xCAT deployment.

Verify:

```bash
xdsh hopper01 'ls -l /etc/xcat/k8s/worker-bootstrap.conf'
xdsh hopper01 'ls -l /usr/local/sbin/xcat-k8s-worker-bootstrap.sh'
xdsh hopper01 'ls -l /usr/local/sbin/xcat-k8s-storage-bootstrap.sh'
xdsh hopper01 'systemctl cat xcat-k8s-storage-bootstrap.service'
xdsh hopper01 'systemctl cat xcat-k8s-worker-bootstrap.service'
```

---

# 14. Per-Node Bootstrap Token

A bootstrap token must be:

- short-lived
- created for the provisioning event
- transferred only when a new registration is actually required
- stored temporarily on the worker
- removed immediately after use
- never baked into the xCAT image

Create a token on the control plane:

```bash
kubeadm token create \
    --ttl 15m \
    --description "xcat-hopper01"
```

Example:

```text
abcdef.0123456789abcdef
```

Do not use:

```bash
kubeadm token create --ttl 0
```

Do not maintain one permanent fleet-wide token.

---

# 15. Secure Token Transfer

Create a temporary file on the xCAT management node:

```bash
TOKEN_FILE="$(mktemp /run/xcat-k8s-token.XXXXXX)"
chmod 0600 "$TOKEN_FILE"
```

Generate the token:

```bash
ssh root@<CONTROL_PLANE_HOST> \
    'kubeadm token create --ttl 15m --description "xcat-hopper01"' \
    > "$TOKEN_FILE"
```

Validate:

```bash
TOKEN="$(cat "$TOKEN_FILE")"

[[ "$TOKEN" =~ ^[a-z0-9]{6}\.[a-z0-9]{16}$ ]] || {
    rm -f "$TOKEN_FILE"
    echo "Invalid token"
    exit 1
}
```

Create the destination directory:

```bash
xdsh hopper01 'install -d -m 0700 -o root -g root /run/xcat-k8s'
```

Transfer:

```bash
xdcp "$TOKEN_FILE" hopper01:/run/xcat-k8s/bootstrap.token
```

Set permissions:

```bash
xdsh hopper01 \
    'chown root:root /run/xcat-k8s/bootstrap.token && chmod 0600 /run/xcat-k8s/bootstrap.token'
```

Delete the management-node copy:

```bash
rm -f "$TOKEN_FILE"
```

Never print the token to the terminal or logs.

---

# 16. Start Worker Bootstrap

On the worker:

```bash
systemctl start xcat-k8s-worker-bootstrap.service
```

Check:

```bash
systemctl status xcat-k8s-worker-bootstrap.service
```

Check state:

```bash
cat /local/k8s/bootstrap/state
```

Expected:

```text
JOINED
```

---

# 17. First-Boot Validation

## 17.1 Storage

```bash
findmnt /local/k8s
findmnt /etc/kubernetes
findmnt /var/lib/kubelet
findmnt /var/lib/containerd
```

All four must be mounted.

## 17.2 Services

```bash
systemctl is-active xcat-k8s-storage-bootstrap.service
systemctl is-active containerd
systemctl is-active xcat-k8s-worker-bootstrap.service
systemctl is-active kubelet
```

Expected:

```text
active
active
active
active
```

## 17.3 Kubernetes identity

```bash
ls -l /etc/kubernetes/kubelet.conf
ls -l /var/lib/kubelet/pki/
```

Inspect certificate:

```bash
openssl x509 \
    -in /var/lib/kubelet/pki/kubelet-client-current.pem \
    -noout \
    -subject \
    -issuer \
    -dates
```

Subject must identify the worker as:

```text
system:node:<worker-name>
```

## 17.4 Bootstrap token cleanup

```bash
test ! -e /run/xcat-k8s/bootstrap.token
```

Expected:

```text
```

with exit code 0.

---

# 18. Control-Plane Validation

Run on the control plane:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

Check nodes:

```bash
kubectl get nodes -o wide
```

Expected:

```text
hopper01   Ready
```

Inspect:

```bash
kubectl describe node hopper01
```

Get the Kubernetes Node UID:

```bash
kubectl get node hopper01 \
    -o jsonpath='{.metadata.uid}{"\n"}'
```

Record this UID before reboot testing.

---

# 19. Normal Reboot Test

Before reboot:

```bash
kubectl get node hopper01
```

Record:

```bash
kubectl get node hopper01 \
    -o jsonpath='{.metadata.uid}{"\n"}'
```

Record certificate:

```bash
openssl x509 \
    -in /var/lib/kubelet/pki/kubelet-client-current.pem \
    -noout -serial -dates
```

Reboot:

```bash
reboot
```

After reboot:

```bash
systemctl is-active xcat-k8s-storage-bootstrap.service
systemctl is-active containerd
systemctl is-active xcat-k8s-worker-bootstrap.service
systemctl is-active kubelet
```

Check state:

```bash
cat /local/k8s/bootstrap/state
```

Expected:

```text
JOINED
```

Check token:

```bash
test ! -e /run/xcat-k8s/bootstrap.token
```

Check kubelet log:

```bash
journalctl -u xcat-k8s-worker-bootstrap.service -b
```

The log must indicate that an existing valid kubelet identity was found.

It must not execute a new `kubeadm join`.

Verify Node UID:

```bash
kubectl get node hopper01 \
    -o jsonpath='{.metadata.uid}{"\n"}'
```

The UID must remain unchanged.

---

# 20. Token Expiration Test

Create a deliberately short-lived test token:

```bash
ssh root@<CONTROL_PLANE_HOST> \
    'kubeadm token create --ttl 60s --description xcat-expiration-test'
```

Wait for expiration:

```bash
sleep 90
```

Check:

```bash
ssh root@<CONTROL_PLANE_HOST> kubeadm token list
```

The expired token must no longer be usable.

Attempting bootstrap with an expired token must result in:

```text
BOOTSTRAP_FAILED
```

The worker must remove:

```text
/run/xcat-k8s/bootstrap.token
```

A new valid token must then be explicitly provisioned before another bootstrap attempt.

---

# 21. Missing Token Test

On a disposable PoC worker with no valid kubelet identity:

```bash
rm -f /run/xcat-k8s/bootstrap.token
```

Run:

```bash
systemctl restart xcat-k8s-worker-bootstrap.service
```

Expected:

```text
BOOTSTRAP_REQUIRED
```

The worker must **not** create its own Kubernetes bootstrap token.

---

# 22. Invalid Token Test

Place an invalid value:

```bash
install -d -m 0700 /run/xcat-k8s
printf '%s\n' 'invalid-token' > /run/xcat-k8s/bootstrap.token
chmod 0600 /run/xcat-k8s/bootstrap.token
```

Run:

```bash
systemctl restart xcat-k8s-worker-bootstrap.service
```

Expected:

```text
BOOTSTRAP_FAILED
```

Verify cleanup:

```bash
test ! -e /run/xcat-k8s/bootstrap.token
```

---

# 23. Persistent State Loss / Recovery Test

Perform this test only on a disposable PoC worker.

First:

```bash
kubectl cordon hopper01
kubectl drain hopper01 \
    --ignore-daemonsets \
    --delete-emptydir-data
```

Stop services:

```bash
systemctl stop kubelet
systemctl stop containerd
```

Do not immediately delete state. Move it aside:

```bash
mv /local/k8s/etc-kubernetes /local/k8s/etc-kubernetes.backup
mv /local/k8s/kubelet /local/k8s/kubelet.backup

mkdir -p /local/k8s/etc-kubernetes
mkdir -p /local/k8s/kubelet
```

Restart storage/bootstrap:

```bash
systemctl restart xcat-k8s-storage-bootstrap.service
systemctl restart xcat-k8s-worker-bootstrap.service
```

Expected state:

```text
BOOTSTRAP_REQUIRED
```

The worker must not automatically generate a token or silently re-register.

---

# 24. Controlled Worker Re-Registration

For a worker that must be intentionally re-registered:

## Control plane

```bash
kubectl cordon hopper01
kubectl drain hopper01 \
    --ignore-daemonsets \
    --delete-emptydir-data
```

Delete the old Kubernetes Node object:

```bash
kubectl delete node hopper01
```

## Worker

Stop services:

```bash
systemctl stop kubelet
systemctl stop containerd
```

Clear the Kubernetes identity only as part of the controlled replacement procedure.

Example:

```bash
rm -rf /local/k8s/etc-kubernetes/*
rm -rf /local/k8s/kubelet/*
```

Start containerd:

```bash
systemctl start containerd
```

Generate a fresh short-lived token on the control plane:

```bash
kubeadm token create \
    --ttl 15m \
    --description "xcat-hopper01-replacement"
```

Transfer the token using the approved xCAT mechanism.

Start:

```bash
systemctl start xcat-k8s-worker-bootstrap.service
```

Validate:

```bash
cat /local/k8s/bootstrap/state
systemctl is-active kubelet
```

Then:

```bash
kubectl get node hopper01 -o wide
```

Expected:

```text
hopper01   Ready
```

---

# 25. Kubelet Certificate Rotation

Do not use repeated:

```bash
kubeadm join
```

for normal kubelet certificate renewal.

Verify kubelet configuration:

```bash
grep -R "rotate-certificates" \
    /var/lib/kubelet /etc/systemd/system/kubelet.service.d \
    2>/dev/null || true
```

Inspect the current certificate:

```bash
openssl x509 \
    -in /var/lib/kubelet/pki/kubelet-client-current.pem \
    -noout \
    -subject \
    -issuer \
    -dates
```

The persistent kubelet state must remain intact so that normal certificate rotation can occur.

---

# 26. xCAT Resynchronization Test

Run:

```bash
updatenode hopper01 -F
```

After synchronization verify:

```bash
ls -l /etc/kubernetes/kubelet.conf
ls -l /var/lib/kubelet/pki/
```

The existing kubelet identity must remain intact.

Verify:

```bash
cat /local/k8s/bootstrap/state
```

Expected:

```text
JOINED
```

No bootstrap token should be added by ordinary xCAT synchronization.

---

# 27. systemd Dependency Validation

Run:

```bash
systemctl list-dependencies kubelet
```

and:

```bash
systemctl list-dependencies containerd
```

Verify the storage service appears in the dependency chain.

Run:

```bash
systemd-analyze critical-chain kubelet.service
```

The expected startup sequence is conceptually:

```text
local filesystem
       |
       v
xcat-k8s-storage-bootstrap
       |
       +----> containerd
       |
       +----> worker bootstrap
                    |
                    v
                  kubelet
```

---

# 28. Failure-Closed Requirements

The implementation must fail closed for:

```text
Missing /local/k8s
Wrong filesystem
Missing persistent mounts
Containerd failure
Missing Kubernetes API configuration
Missing CA hash
Malformed bootstrap token
Expired bootstrap token
kubeadm join failure
Missing kubelet identity after join
```

In these conditions the system must not:

```text
format the SSD automatically
delete valid Kubernetes identity
generate an unlimited token
run kubeadm join on every reboot
skip CA verification
copy kubelet private keys from xCAT
persist the bootstrap token
print the bootstrap token in logs
```

---

# 29. Fleet Rollout

For 100+ workers, do not reboot or bootstrap the entire fleet simultaneously.

Use controlled batches.

Example batch:

```text
10 workers
    |
    v
validate
    |
    v
next 10 workers
```

For each batch validate:

```bash
kubectl get nodes -o wide
```

Check for non-Ready nodes:

```bash
kubectl get nodes --no-headers \
    | awk '$2 != "Ready" {print}'
```

Check worker bootstrap state:

```bash
xdsh <node-range> 'cat /local/k8s/bootstrap/state'
```

Expected:

```text
JOINED
```

---

# 30. GPU Worker Validation

For GPU workers:

```bash
nvidia-smi
```

Verify Kubernetes allocatable GPU resources:

```bash
kubectl describe node hopper01 \
    | grep -A20 -i Allocatable
```

Verify the NVIDIA device plugin:

```bash
kubectl get pods -A | grep -i nvidia
```

The xCAT/Kubernetes bootstrap mechanism must not modify NVIDIA driver state or GPU configuration during normal Kubernetes worker reboots.

---

# 31. Production PoC Acceptance Checklist

| Test | Expected result |
|---|---|
| Fresh worker boot | Storage mounted |
| First bootstrap | Exactly one `kubeadm join` |
| Worker becomes Ready | PASS |
| Token transferred securely | PASS |
| Token removed after join | PASS |
| `kubelet.conf` persisted | PASS |
| kubelet client certificate persisted | PASS |
| Worker reboot | No second `kubeadm join` |
| Node UID after reboot | Unchanged |
| Kubelet after reboot | Ready |
| Containerd after reboot | Active |
| Missing token | `BOOTSTRAP_REQUIRED` |
| Expired token | `BOOTSTRAP_FAILED` |
| Invalid token | `BOOTSTRAP_FAILED` |
| Join failure | Token deleted |
| Persistent identity loss | Controlled recovery required |
| xCAT resync | Kubernetes identity unchanged |
| Wrong filesystem | Bootstrap fails closed |
| Missing storage | Bootstrap fails closed |
| GPU worker | GPU remains available |
| Fleet batch | No bootstrap storm |

---

# 32. Final Production State

After successful PoC completion, each worker should have:

```text
/etc/xcat/k8s/worker-bootstrap.conf
/usr/local/sbin/xcat-k8s-storage-bootstrap.sh
/usr/local/sbin/xcat-k8s-worker-bootstrap.sh

/etc/systemd/system/
├── xcat-k8s-storage-bootstrap.service
├── xcat-k8s-worker-bootstrap.service
├── kubelet.service.d/
│   └── 10-xcat-bootstrap.conf
└── containerd.service.d/
    └── 10-xcat-storage.conf

/local/k8s/
├── etc-kubernetes/
│   └── kubelet.conf
├── kubelet/
│   └── pki/
├── containerd/
└── bootstrap/
    └── state
```

The transient bootstrap credential exists only during a controlled registration event:

```text
/run/xcat-k8s/bootstrap.token
```

After successful or failed bootstrap:

```text
/run/xcat-k8s/bootstrap.token
        |
        v
      deleted
```

Normal lifecycle:

```text
xCAT provisions OS
        |
        v
Persistent local Kubernetes state mounted
        |
        v
Does valid kubelet identity exist?
        |
     +--+--+
     |     |
    YES    NO
     |     |
     |    obtain short-lived token
     |     |
     |    kubeadm join
     |     |
     |    create persistent identity
     |     |
     +-----+
        |
        v
      kubelet
        |
        v
 Kubernetes Node Ready
        |
        v
 Reboot
        |
        v
 Existing identity reused
        |
        v
 No kubeadm join
```

---

# 33. Required Operational Rule

The fundamental production rule for this PoC is:

> **xCAT owns provisioning and node lifecycle orchestration; Kubernetes owns worker identity and cluster membership.**

Therefore:

```text
xCAT:
  OS image
  packages
  configuration
  bootstrap orchestration

Kubernetes:
  Node object
  kubelet identity
  kubelet certificates
  cluster membership
  workload state
```

A normal reboot is therefore **not a new Kubernetes registration event**.

A new `kubeadm join` is performed only during an explicitly authorized initial registration or worker replacement/recovery procedure.

---

# 34. Official References

Kubernetes kubeadm join:
https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/

Kubernetes bootstrap tokens:
https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/

Kubernetes kubeadm token:
https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/

Kubernetes kubelet certificate rotation:
https://kubernetes.io/docs/tasks/tls/certificate-rotation/

xCAT documentation:
https://xcat-docs.readthedocs.io/

xCAT synclist:
https://xcat-docs.readthedocs.io/en/stable/advanced/syncfiles/

---

# 35. PoC Completion Criteria

The PoC is considered successful only when all of the following are demonstrated on at least one CPU worker and one GPU worker:

```text
[PASS] xCAT provisions worker
[PASS] Local persistent storage is mounted
[PASS] Kubernetes state survives reboot
[PASS] First boot performs controlled kubeadm join
[PASS] Bootstrap token is short-lived
[PASS] Bootstrap token is not stored in the image
[PASS] Bootstrap token is deleted after use
[PASS] CA pinning is enabled
[PASS] Worker becomes Ready
[PASS] Reboot does not perform kubeadm join
[PASS] Node UID remains unchanged after reboot
[PASS] Kubelet certificate rotation remains possible
[PASS] Missing identity enters controlled recovery
[PASS] Expired token fails safely
[PASS] Invalid token fails safely
[PASS] xCAT resync does not overwrite Kubernetes identity
[PASS] GPU resources remain available
[PASS] Fleet rollout can be performed in controlled batches
```

**End of production PoC implementation runbook.**
