# WHAT?

The problem we are facing in our HPC Environment is that any compute node / GPU node in the cluster to be added as a k8s worker node connected to the control-plane of the cluster, is a stateless machine that does not have a persistent data. Even if we put the kubeadm worker node joining command on these rebooted nodes, the nodes will have a new identity. That will create a confusion like 6 nodes restarted once will result in 12 distinct identities.

# WHY?

When a stateless `xCAT` node boots, it starts with a completely blank canvas in RAM. [1, 2]
1. When you run `kubeadm join`, Kubernetes generates a unique node identity, crypto keys, and certificates, placing them into local directories like `/etc/kubernetes/` and `/var/lib/kubelet/`.
2. When the node reboots, **those credentials are wiped out** because they reside in RAM. [1, 2]
3. On the next boot, the node executes `kubeadm join` again. Because its old credentials are gone, the Kubernetes Control Plane treats it as a **brand-new node**, resulting in `node-1`, `node-1-f8df...`, etc., accumulating indefinitely.

### In one line

> **The root cause is that the xCAT stateless setup removes the Kubernetes worker identity when the node reboots, so the node cannot come back as the same Kubernetes worker.**
> 

And **that** is why you need an automatic bootstrap/rejoin mechanism when the node starts again.

# Solution?

There is a solution to this case as: 

1. We can install the OS in stateful mode and we can use the same machine without loosing the identity.
**BUT** THIS IS NOT OUR REQUIREMENT AND IT IS HARD TO MANAGE STATEFUL NODES.
2. STATELITE mode persisting the required directories.
3. Using xcat localdisk feature with stateless boot over pxe using xcat but requires the bootstrap script that checks if the data is on the localdisk or not if yes then add the node to the k8s cluster and if no then run the joining command to join this node as k8s cluster worker node.
4. The LFS configured in such a way that nodes have access to specific directory based on their nodename or ip that persists the required data as required by the k8s cluster control-plane or kubelet, whatever! JUST LIKE OUR HOME DIRECTORY.
BUT THIS IS NOT PRODUCTION-GRADE SOLUTION BECAUSE IT HAS DEPENDANIES LIKE NETWORKING AND CONSISTENT FS.

---

# About Tokens

The kubeadm join token is strictly used for initial authentication and bootstrap TLS client certificate generation during the kubeadm join phase.

## 1. Token Expiration Concern

> The Kubernetes bootstrap token has a **24-hour TTL**. Once the token expires, it is automatically deleted and can no longer be used to join nodes to the Kubernetes Control Plane.
If `kubeadm join <token>` is included in the xCAT stateless OS image boot script, a node can successfully join the Control Plane only while the token is valid. **After 24 hours, if the node restarts or reboots, the expired token will prevent it from rejoining the Kubernetes Control Plane.**
> 

## 2. Solution:

1. Create a Token that never expires

```
sudo kubeadm token create ---ttl 0 

#for a custom lifetime
sudo kubeadm token create --ttl 48h
```

Note: Infinite tokens are handy for automation but present a security risk if exposed.

*From worker side:  Never. Once a worker node has successfully joined the cluster, it never loses communication due to the token expiring.*

- How Communication Works Post-Join
    - **TLS Bootstrap:** The temporary token is only used to talk to the control plane long enough to submit a Certificate Signing Request (CSR).
    - **Kubelet Client Certificate:** The control plane signs this request and issues a unique client certificate to the worker node's `kubelet`.
    - **Automatic Renewal:** By default, the `kubelet` automatically renews its own unique certificate before it expires (usually every 1 year). It does not need the original bootstrap token ever again.
- Why Else Might a Node Lose Communication?
    
    If your worker node is losing communication with the control plane, it is likely due to one of these common issues:
    
    - **Network Connectivity:** Firewalls, security groups, or a broken network plugin (Calico, Flannel, etc.) are blocking ports `6443` (API Server) or `10250` (Kubelet).
    - **Kubelet Crashed:** The `kubelet` service on the worker node stopped running. Check its status using `systemctl status kubelet`.
    - **Expired Kubelet Certificate:** If the node was powered off for many months, its auto-renewal might have missed the window, requiring a manual certificate renewal.

---

# About files and node identity

- 1. What is stored in `/etc/kubernetes/` ?
    
    This contains important Kubernetes node configuration and credentials.
    
    For a joined worker, you'll typically have things such as:
    
    ```
    /etc/kubernetes/
    ├── kubelet.conf      #  tells the kubelet how to authenticate to and communicate with the Kubernetes API server
    ├── kubeadm-flags.env # env variables
    └── pki/              # Has certificates
    ```
    
    The most important one is:
    
    ```
    /etc/kubernetes/kubelet.conf
    ```
    
    It tells the kubelet how to authenticate to and communicate with the Kubernetes API server.
    
- 2. What is stored in  `/var/lib/kubelet/` ?
    
    This is even more important for a worker.
    
    It contains kubelet runtime state, configuration, certificates, pod state, plugins, and other information.
    
    For example:
    
    ```
    /var/lib/kubelet/
    ├── config.yaml
    ├── pki/
    │   ├── kubelet-client-current.pem
    │   └── kubelet.crt
    ├── pods/
    ├── plugins/
    ├── plugins_registry/
    └── checkpoints/
    ```
    
    The exact contents depend on Kubernetes/kubelet version and configuration.
    
- 3. What is stored in  `/var/lib/containerd/` ?
    
    This contains container runtime state, including image/content/snapshot information.
    
    Think:
    
    ```
    /var/lib/containerd/
           |
           +-- images
           +-- content
           +-- snapshots
           +-- container metadata
    ```
    
    If this is stateless, the node loses its locally cached container images and runtime state when the OS is recreated.
    
    Basically it is cache for images.
    

---

The **loophole** is therefore:

> **xCAT's persistent local disk can preserve the worker's Kubernetes information across reboots, but it does not solve the initial joining of a new worker. The first join is still a manual step.**
> 

So the remaining problem is to **automate only that first join**, while leaving already-joined workers untouched when they reboot.

---

# **What if someone steals the token?**

Consider:

```
GPU-worker-17
     │
     │ token accidentally leaked
     ▼
attacker-controlled machine
     │
     │ can reach 10.x.x.x:6443
     ▼
Kubernetes API
```

The attacker can attempt to use that token for bootstrap authentication.

That's why the token should be:

- **short-lived**
- not shared unnecessarily
- not stored permanently on workers
- not embedded in images
- not committed to Git
- not exposed in logs
