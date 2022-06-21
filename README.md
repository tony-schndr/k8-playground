# Kubernetes base install with kubeadm

This project uses vagrant, libvirt, and ansible to bootstrap 3 VMs and install
the base configuration needed to use kubeadm command to create a 1 master 2 worker
kubernetes cluster.

Start the virtual machines
`vagrant up`

Once virtual machines are up, ssh into the primary controller with
`vagrant ssh controller-1`

Then use kubeadm init to bootstrap the control plane
```
kubeadm init --pod-network-cidr 10.32.0.0/16 \
  --apiserver-advertise-address 192.168.1.21
```

Set kubeconfg to start using the cluster
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Join the worker nodes, ssh into `worker-1` and `worker-2` using `vagrant ssh`, become root with `sudo -i`, and run kubeadm join command that was output during kubeadm init
```
kubeadm join 192.168.1.21:6443 --token owmpxs.7a4gv8o87do0aw3w \
        --discovery-token-ca-cert-hash sha256:a1e8201bd0975e6d1581b82111b970b9b4ad5bf3278f9734f3428eb8d5b4e487
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
Backup / Restore of etcd
#### Take a snapshot of the etcd cluster
NOTE: Tested with a single control plane node.
```
kubectl -n kube-system exec -it etcd-controller-1 -- sh -c "ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key --cacert=/etc/kubernetes/pki/etcd/ca.crt snapshot save /var/lib/etcd/backup.db"
```

#### Restore etcd cluster from snapshot.
Stop the kube-apiserver before restore, if the kube-apiserver is deployed via kubelet manifest, move the manifest.
Then use crictl to exec commands, find the etcd container `crictl ps | grep etcd`
```
crictl exec -it 87351a93bc4a6 sh -c "etcdctl snapshot restore /var/lib/etcd/backup.db --data-dir /var/lib/etcd/backup"
```
Stop etcd, move the manifest like with kube-apiserver

Move the newly restored etcd data directory back into place

```
rm -rf /var/lib/etcd/member
mv /var/lib/etcd/backup/member /var/lib/etcd
```

Move the manifests back to start etcd and api-server
```
mv /etc/kubernetes/*.yaml /etc/kubernetes/manifests/
```