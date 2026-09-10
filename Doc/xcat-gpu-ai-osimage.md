
Step1: Write the bootstrap script for kubernetes persistent storage and to join worker node to k8s cluster

1.1 copy the storage-bootstrap script in osimage at ../rootimg/usr/local/sbin/
```bash
#gpu-ai osimage rootimgdir
export rootimgdir=/install/netboot/rocky9.6/x86_64/gpu-ai
# Install the k8s-worker.tgz into your machine
tar -xvf k8s-worker.tgz
cd k8s-worker
cp -ar bootstrap_script/xcat-k8s-storage-bootstrap.sh $rootimgdir/rootimg/usr/local/sbin/
chmod 700 $rootimgdir/rootimg/usr/local/sbin/xcat-k8s-storage-bootstrap.sh
chown -R root:root $rootimgdir/rootimg/usr/local/sbin/xcat-k8s-storage-bootstrap.sh
```

1.2 copy the worker-bootstrap script in osimage at ../rootimg/usr/local/sbin/
```bash
cp -ar bootstrap_script/xcat-k8s-worker-bootstrap.sh $rootimgdir/rootimg/usr/local/sbin/
chmod 700 $rootimgdir/rootimg/usr/local/sbin/xcat-k8s-worker-bootstrap.sh
chown -R root:root $rootimgdir/rootimg/usr/local/sbin/xcat-k8s-worker-bootstrap.sh
```

Step2: Write the k8s worker node join related configuration file

```bash
cp k8s_worker_conf/worker-bootstrap.conf $rootimgdir/rootimg/etc/xcat/k8s/worker-bootstrap.conf
chmod 600 $rootimgdir/rootimg/etc/xcat/k8s/worker-bootstrap.conf
chown -R root:root $rootimgdir/rootimg/etc/xcat/k8s/worker-bootstrap.conf
```

Step3: Write the bootstrap systemd service to run the bootstrap script
```bash
cp -ar systemd_service/k8s-storage-bootstrap.service  $rootimgdir/rootimg/etc/systemd/system/
cp -ar systemd_service/k8s-worker-bootstrap.service $rootimgdir/rootimg/etc/systemd/system/
chmod 700 $rootimgdir/rootimg/etc/systemd/system/k8s-storage-bootstrap.service
chmod 700 $rootimgdir/rootimg/etc/systemd/system/k8s-worker-bootstrap.service
chown root:root $rootimgdir/rootimg/etc/systemd/system/k8s-storage-bootstrap.service
chown root:root $rootimgdir/rootimg/etc/systemd/system/k8s-worker-bootstrap.service
```

Step4: Enable the bootstrap services
```bash
systemctl daemon-reload
systemctl enable k8s-storage-bootstrap.service k8s-worker-bootstrap.service

```


Step5: Make sure token id file must be in synclist
```bash
# copy the join worker token from control plane into this file
#using the following command
$ kubeadm token list
or
$cat /etc/kubernetes/pki/ca.crt \
  | openssl x509 -pubkey \
  | openssl rsa -pubin -outform der 2>/dev/null \
  | openssl dgst -sha256 -hex

#Copy the above token in this file
vim /xcatdata/token/bootstrap.token

#Add the synclist path in sysnclist file
vim $(lsdef -t osimage rocky9.6-x86_64-netboot-gpu-ai | awk  -F '=' '/synclists/ {print $2}'  )
/xcatdata/worker-bootstrap.conf -> /etc/xcat/k8s/worker-bootstrap.conf
/xcatdata/token/bootstrap.token -> /run/xcat-k8s/bootstrap.token
```
