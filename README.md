# Kubernetes base install of kubeadm

This project uses vagrant, vbox, and ansible to bootstrap 3 VMs and install
the base configuration needed to use kubeadm command.

Start the virtual machines
`vagrant up`

Once virtual machines are up, ssh into the primary controller with
`vagrant ssh controller-1`

Then use kubeadm init to bootstrap the control plane
```
kubeadm init --pod-network-cidr 172.18.0.0/16 \
  --apiserver-advertise-address 10.240.0.21
```