# Kubernetes base install of kubeadm

This project uses vagrant, vbox, and ansible to bootstrap 3 VMs and install
the base configuration needed to use kubeadm command.

Start the virtual machines
`vagrant up`

Once virtual machines are up, ssh into the primary controller with
`vagrant ssh controller-1`

Then use kubeadm init to bootstrap the control plane
```
kubeadm init --pod-network-cidr 10.32.0.0/16 \
  --apiserver-advertise-address 10.240.0.21
```

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
update workers kubelet config to add --node-ip, this fixes "kubectl exec return error: unable to upgrade connection: pod does not exist" https://github.com/kubernetes/kubernetes/issues/63702

vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
Environment="KUBELET_EXTRA_ARGS=--node-ip=10.240.0.32"


helpful network debugging pod

```
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
```

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