# ansible-azure
Ansible samples: Azure dynamic inventory, ARM Template, Packer

# Provision the infrastructure on Azure

First, we can create 3 VMs, virtual network, availability set and load balancer on Azure using Ansible with ARM Tempalte. Ansible support azure resouce manager but it doesen't implement all the new resource type. So It is better to use ARM template. It include LB configuratoin and enable WinRM with custom script extension.

Setting up tags is important if you want to use dynamic inventory. I'm using tags to seperate creating and destroying.

```
$ cd ansible
$ ansible-playbook provision.yml --tags create
```

