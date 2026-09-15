

# The Problem

 We have a large HPC environment where worker machines are created from a **common, stateless xCAT image**. Whenever a worker starts, it should automatically become part of our Kubernetes cluster and be ready to run workloads.

 Currently, adding a new worker requires manually configuring the machine and running the Kubernetes registration command. This does not scale when we have many workers. It also becomes difficult to manage when machines are frequently rebooted, reprovisioned, or replaced.

---

 > The challenge is to design a **secure and fully automated way for a newly booted worker to identify itself, obtain the information it needs to connect to the Kubernetes cluster, and register itself automatically**, without storing permanent credentials or another worker's identity inside the shared image.

---

 # Why Is This Happening?

 ## The Root Cause

 When a worker joins Kubernetes, Kubernetes creates files on that machine that allow it to **remain recognized as the same worker**.

 When the machine reboots, those files are lost because the machine starts again from the original xCAT image.

 ### Why This Happens

 The Kubernetes worker's identity is stored **on the local machine**, but the xCAT setup does not preserve those files across a reboot.

 In simple terms:

 > **The machine's identity is not persistent.**

 As a result, when the machine reboots, it loses its identity and state.

 From Kubernetes' point of view, it can then look like:

 > "A new machine is trying to join."

 ### In One Line

 > **The root cause is that the xCAT stateless setup removes the Kubernetes worker's identity when the node reboots, so the node cannot come back as the same Kubernetes worker.**

 And **that** is why an automatic bootstrap/rejoin mechanism is needed when the node starts again.

---

 # About Tokens

 The `kubeadm join` token is used only during the initial authentication and bootstrap process when a worker joins the Kubernetes cluster.

 ## 1\. Token Expiration Concern

 > The Kubernetes bootstrap token has a **24-hour TTL**. Once the token expires, it is automatically deleted and can no longer be used to join a node to the Kubernetes control plane.
>
>  If `kubeadm join <token>` is included in the xCAT stateless OS image boot script, a node can successfully join the control plane only while the token is valid. **After 24 hours, if the node restarts or reboots, the expired token will prevent it from rejoining the Kubernetes control plane.**

 ## 2\. Solution

 One option is to create a token that does not expire:

```
sudo kubeadm token create --ttl 0
```

 For a custom lifetime:

```
sudo kubeadm token create --ttl 48h
```

 **Note:** Tokens that never expire can be useful for automation, but they create a security risk if the token is exposed.

 ### From the Worker's Side: The Token Is Not Needed After the Initial Join

 Once a worker has successfully joined the cluster, it does **not** lose communication with the control plane simply because the bootstrap token expires.

 ### How Communication Works After the Worker Joins

 - **Initial TLS bootstrap:** The temporary token is used to authenticate with the control plane and start the certificate process.
- **Kubelet client certificate:** The control plane approves the request and provides the worker's `kubelet` with a unique client certificate.
- **Automatic renewal:** By default, the `kubelet` automatically renews its own certificate before it expires, usually within its certificate lifetime. It does not need the original bootstrap token again.

 ### Why Else Might a Node Lose Communication?

 If a worker loses communication with the control plane, it is likely due to one of these common issues:

 - **Network connectivity:** Firewalls, security groups, or a broken network plugin such as Calico or Flannel may be blocking ports `6443` (API server) or `10250` (Kubelet).
- **Kubelet stopped:** The `kubelet` service on the worker may have stopped. Its status can be checked with:

  ```
  systemctl status kubelet
  ```
- **Expired Kubelet certificate:** If the node has been powered off for many months, automatic certificate renewal may have been missed, requiring manual certificate renewal.

---

 # The Solution With a Loophole

 ### Using xCAT's Local Disk Feature

 We can solve the problem of losing the worker's identity after a reboot by using **xCAT's local disk feature**.

 A part of the worker's disk can be kept **persistent**, even though the rest of the operating system is recreated from the xCAT image.

 This allows us to save the Kubernetes information on the persistent part of the disk.

```
Worker boots
     ↓
xCAT loads the operating system
     ↓
Persistent disk is mounted
     ↓
Previous Kubernetes information is available
     ↓
Worker is recognized as the same worker
```

 This means that when an existing worker reboots, we **do not need to connect it to Kubernetes again**. Its previous information is still available on the persistent disk.

 ### The Loophole

 The problem is solved **only after the worker has already joined Kubernetes once**.

 For a completely new worker, the persistent disk does not contain any Kubernetes information yet.

 Therefore, the first time the worker starts, we still have to manually run:

```
kubeadm join ...
```

 After that, the information created during the join can be saved on the persistent disk.

 The process then becomes:

```
New worker
    ↓
No Kubernetes information
    ↓
Manually run kubeadm join
    ↓
Kubernetes information is created
    ↓
Save it on persistent disk
    ↓
Future reboots use the saved information
```

 The **loophole** is therefore:

 > **xCAT's persistent local disk can preserve the worker's Kubernetes information across reboots, but it does not solve the initial joining of a new worker. The first join is still a manual step.**

 So the remaining problem is to **automate only that first join**, while leaving already-joined workers untouched when they reboot.

 > **This is why the Kubernetes cluster connection is broken when the node restarts. Even if the worker's data is persisted, a completely new worker still cannot join the cluster without the initial join process.**

---

 # Security Report

 ## What If Someone Steals the Token?

 Consider the following situation:

```
GPU-worker-17
     │
     │ token accidentally leaked
     ▼
Attacker-controlled machine
     │
     │ can reach 10.x.x.x:6443
     ▼
Kubernetes API
```

 The attacker could attempt to use the token for bootstrap authentication.

 That's why the token should be:

 - **Short-lived**
- Not shared unnecessarily
- Not stored permanently on workers
- Not embedded in images
- Not committed to Git
- Not exposed in logs

---

 # The Conclusion

 A worker node that needs to be added to the cluster after a reboot needs to use the `kubeadm join` process.

 But, the `kubeadm join` command cannot simply be placed in the shared xCAT image and expected to work indefinitely because the bootstrap token has a limited lifetime.

 We have therefore set up a **bootstrap service on the compute node**. Whenever a machine boots using the given image, the service will use the available token to handle the Kubernetes join process automatically.

 The remaining part of the solution is to make this bootstrap process secure and ensure that it can distinguish between:

 - A **new worker** that needs to join Kubernetes for the first time.
- An **existing worker** that already has its Kubernetes information stored on the persistent disk and only needs to restore that information after reboot.

 This allows us to automate the initial worker registration while preserving the existing worker identity across reboots.

