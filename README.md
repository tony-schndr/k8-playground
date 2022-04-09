# Kubernetes base install of kubeadm

This project uses vagrant, vbox, and ansible to bootstrap 3 VMs and install
the base configuration needed to use kubeadm command.

Start the virtual machines
`vagrant up`

IF you run into an ACCESSDENIED error when setting up networking similar to below:
```
There was an error while executing `VBoxManage`, a CLI used by Vagrant
for controlling VirtualBox. The command and stderr is shown below.

Command: ["hostonlyif", "ipconfig", "vboxnet1", "--ip", "10.240.0.1", "--netmask", "255.255.255.0"]

Stderr: VBoxManage: error: Code E_ACCESSDENIED (0x80070005) - Access denied (extended info not available)
VBoxManage: error: Context: "EnableStaticIPConfig(Bstr(pszIp).raw(), Bstr(pszNetmask).raw())" at line 242 of file VBoxManageHostonly.cpp
```
THEN create the file `/etc/vbox/networks.conf` and add the IP range used by the vms 
```
* 10.240.0.0/16
```

Once virtual machines are up, ssh into the primary controller with
`vagrant ssh controller-1`

Then use kubeadm init to bootstrap the control plane
```
kubeadm init --pod-network-cidr 10.32.0.0/16 \
  --apiserver-advertise-address 10.240.0.21
```

Set kubeconfg to start using the cluster
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Join the worker nodes, ssh into `worker-1` and `worker-2` using `vagrant ssh`, become root with `sudo -i`, and run kubeadm join command that was output during kubeadm init
```
kubeadm join 10.240.0.21:6443 --token qt6rqz.xw97h2qodnaw03t9 \
        --discovery-token-ca-cert-hash sha256:32e0062f38a44602c99e54fe010b8708a03f178f3dec55644d001fe8ba33b54e 
```

Apply networking plugin
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

Smoke test
---

Create a deployment and scale it to 2 + pods, check that pods are deployed between both workers
```
root@controller-1:~# kubectl create deployment nginx --image=nginx:latest
deployment.apps/nginx created

root@controller-1:~# kubectl scale deployment nginx --replicas=5
deployment.apps/nginx scaled

root@controller-1:~# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
nginx-7c658794b9-5fl2x   1/1     Running   0          44s   10.32.0.5   worker-1   <none>           <none>
nginx-7c658794b9-87s8p   1/1     Running   0          89s   10.44.0.1   worker-2   <none>           <none>
nginx-7c658794b9-9qg8g   1/1     Running   0          44s   10.32.0.4   worker-1   <none>           <none>
nginx-7c658794b9-hn7dq   1/1     Running   0          44s   10.44.0.3   worker-2   <none>           <none>
nginx-7c658794b9-nn54d   1/1     Running   0          44s   10.44.0.2   worker-2   <none>           <none>
```

Expose the nginx deployment
```
root@controller-1:~# kubectl expose deployment  nginx --target-port 80 --port 8080
service/nginx exposed
```

Check network connectivity, run this pod that has network troubleshooting tools installed. 
```
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
```

Once at the bash prompt in the helper pod, curl the nginx service, validate html is returned.
```
bash-5.1# curl nginx:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Autoscaling
---
Horizontal pods autoscaling requires metric server, install it with

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Metrics server will not start as it requires IP SANs added to node certs OR
edit the metrics server deployment and pass `--kubelet-insecure-tls`
```yaml
  containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        image: k8s.gcr.io/metrics-server/metrics-server:v0.6.1

```

ETCD
---
Run etcd commands using etcdctl, if etcd is deployed via pod on the controller use kubectl exec $etcd_pod_name -n kube-system -- <etcdctl command here>
```
# take a snapshot of the etcd cluster
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key --cacert=/etc/kubernetes/pki/etcd/ca.crt snapshot save /var/lib/etcd/backup.db

```
```
# Restore etcd cluster from snapshot

```