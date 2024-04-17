# Build a Kubernetes cluster using K3S via Ansible

## K3s Ansible Playbook

Build a Kubernetes cluster using Ansible with k3s.
The goal is easily to install a Kubernetes cluster on machines.

Reference provisioner is a local Vagrant VM setup

## System requirements

Deployment environment must have Ansible 2.4.0+
Server-nodes and agent-nodes must have passwordless SSH access (Terraform enables this)

## Usage

### Install Local with Vagrant:

This needs some RAM and CPU.
Do not try this with less than 6GB of free RAM.
The Ansible run is split up into two parts.
One run on Vagrant provision time, and the second need to run separately.

This is because in Vagrant every machine is provisioned on its own in random order.
But to install K3S order matters.

- [Install Vagrant](https://www.vagrantup.com/docs/installation)
- Install `vagrant-scp` plugin: ```vagrant plugin install vagrant-scp```
- Install Ansible i.e. with `apt install ansible`
- Start the cluster `vagrant up`
- Install additional Ansible Role from Galaxy `ansible-galaxy install -r collections/requirements.yml`
- Edit `ansible.cfg` and ensure the inventory for vagrant is used.
- Install K3S and prepare Helm: `ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory install.yml`
  This installs all
- Copy kubeconfigs to your local machine: `ansible-playbook copy-kube-configs.yml`

### IP-Addresses

Local IP-addresses are

- `172.16.0.20` for initial manager cluster-manager-1
- `172.16.0.3x` for agent-x

## Access Vagrant Servers

```shell
vagrant ssh cluster-[manager|agent]-x
```

### Uninstall

Run `vagrant destroy -f`

# Access tools of cluster

Append a line to your `/etc/hosts` (as root user):

```shell
echo "172.16.0.20	portainer.cluster traefik.cluster" >>/etc/hosts
```

Then access the tools in the browser using

- `https://portainer.cluster`

If you were too slow with Portainer to set a password, restart the pod with `kubectl rollout restart deployment portainer -n tools-system` and hurry up.


# Troubleshooting libvirt/qemu

In `virtmanager` you can delete the virtual machines.

For some reason networks and volumes in libvirt are kept.
To clean this up those can be deleted with `virsh`:

To clean up volumes: first, get the volumes from the default pool:
```
virsh vol-list default

 Name                                                                     Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 debian-VAGRANTSLASH-bullseye64_vagrant_box_image_11.20220328.1_box.img   /var/lib/libvirt/images/debian-VAGRANTSLASH-bullseye64_vagrant_box_image_11.20220328.1_box.img
 generic-VAGRANTSLASH-debian11_vagrant_box_image_4.1.0_box.img            /var/lib/libvirt/images/generic-VAGRANTSLASH-debian11_vagrant_box_image_4.1.0_box.img
 k3s-ansible_cluster-agent-0.img                                          /var/lib/libvirt/images/k3s-ansible_cluster-agent-0.img
 k3s-ansible_cluster-server-0.img                                         /var/lib/libvirt/images/k3s-ansible_cluster-server-0.img
 k3s-ansible_cluster-server-1.img                                         /var/lib/libvirt/images/k3s-ansible_cluster-server-1.img

```

Then for each VARIANT volume starting with `k3s-ansible` remove do:

```
virsh vol-delete --pool default k3s-ansible_cluster-VARIANT.img
```

To clean up networks, first list the existing ones:

```
virsh net-list

 Name                      State    Autostart   Persistent
------------------------------------------------------------
 default                   active   yes         yes
 k3s-ansible0              active   no          yes
 vagrant-libvirt           active   no          yes

```

Then delete the network `k3s-ansible0`.

```
virsh net-delete k3s-ansible0
```

Now Vagrant should work again as expected.


## Debug container with Ubuntu

For debugging purposes, an `Ubuntu container](https://artifacthub.io/packages/helm/open-charts/ubuntu) inside cluster can be helpful.

```shell
helm install my-ubuntu open-charts/ubuntu -n cluster --version 1.2.1
```
see https://artifacthub.io/packages/helm/open-charts/ubuntu for details and newer versions.

## Reset K3S

**all data is lost**

On all machines do:

- `sudo systemctl disable k3s` (on manager nodes)
- `sudo systemctl disable k3s-node` (on agent nodes)
- `sudo reboot` (otherwise not all processes are stopped)
- (login via ssh again)
- `sudo rm -rf /etc/rancher/k3s /run/k3s /run/flannel /var/lib/rancher/k3s /var/lib/kubelet`
- `sudo systemctl enable k3s` (on manager nodes)
- `sudo systemctl enable k3s-node` (on manager nodes)
- `systemctl start k3s` (on manager nodes)
- `systemctl start k3s-node` (on agent nodes)

Then run the ansible playbook install as described above.

# Contributions

Base inspiration by [@itwars](https://github.com/itwars) - base source <https://github.com/itwars/k3s-ansible>

Modifications by [Jens Klein](https://github.com/jensens), A.Loechel, P.Auersperg, et al
