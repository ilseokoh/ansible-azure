# ansible-azure
Ansible + Azure samples: Azure dynamic inventory, ARM Template

# Provision the infrastructure on Azure

First, we can create 3 VMs, virtual network, availability set and load balancer on Azure using Ansible with [ARM Tempalte](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-overview). Ansible support azure resouce manager to create resources but it doesen't implement all the new resource type at this point. So It is better to use ARM template. 

![System diagram](/img/azure-provisoin.jpg)

2 check-points of [armtemplae/azuredeploy.json](/armtemplate/azuredeploy.json)
1. Each VM has it's own tag. (service:web, service:database) It's useful when we use dynamic inventory. You can set a filter with tags. 
1. Excute powershell script with "custom script extention". [The script](https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1) enables  WinRM.

Prerequisite for this command. 
1. [Use portal to create an Azure Active Directory application and service principal that can access resources](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-create-service-principal-portal)
1. make credential file on your "Ansible controller". The default location is '~/.azure/credential' You can see [this document](http://docs.ansible.com/ansible/devel/scenario_guides/guide_azure.html#storing-in-a-file). 
1. Configure your virtual network to communication between Ansible controller and generated VMs. I setup Virtual network peering between 2 VNet. 

```
$ cd ansible
$ ansible-playbook provision.yml --tags create
```

# Run ansible playbook with static inventory

Prerequisite for this command. 
1. Make your own [inventory.yml](/ansible/inventory.yml) file with VM's private ip.
1. Take a look at [group_vars/all.yml](/group_vars/all.yml) for WinRM config

### Run webserver playbook
```
$ ansible-playbook -i inventory.yml webservers.yml

PLAY [web] *********************************************************************************************

TASK [Gathering Facts] *********************************************************************************
ok: [10.10.0.5]
ok: [10.10.0.4]

TASK [common : Install Chocolatey] *********************************************************************
changed: [10.10.0.5]
changed: [10.10.0.4]

TASK [common : Ensure .NET Framework 4.7.1 is installed using chocolatey] ******************************
ok: [10.10.0.4]
ok: [10.10.0.5]

TASK [webserver : Ensure IIS webserver is installed] ***************************************************
ok: [10.10.0.4]
ok: [10.10.0.5]

TASK [webserver : Deploy default iisstart.htm file] ****************************************************
ok: [10.10.0.4]
ok: [10.10.0.5]

TASK [webserver : Ensure IIS service is running] *******************************************************
ok: [10.10.0.4]
ok: [10.10.0.5]

PLAY RECAP *********************************************************************************************
10.10.0.4                  : ok=6    changed=1    unreachable=0    failed=0
10.10.0.5                  : ok=6    changed=1    unreachable=0    failed=0

```

### Run database playbook with static inventory
```
$ ansible-playbook -i inventory.yml databases.yml

PLAY [db] **********************************************************************************************

TASK [Gathering Facts] *********************************************************************************
ok: [10.10.1.4]

TASK [common : Install Chocolatey] *********************************************************************
changed: [10.10.1.4]

TASK [common : Ensure .NET Framework 4.7.1 is installed using chocolatey] ******************************
ok: [10.10.1.4]

TASK [database : install visual c++ redistributable 2013] **********************************************
ok: [10.10.1.4]

TASK [database : Install mysql database] ***************************************************************
ok: [10.10.1.4]

TASK [database : Firewall rule to allow MySQL on TCP port 3306] ****************************************
ok: [10.10.1.4]

TASK [database : Install MySQL as a windows service] ***************************************************
changed: [10.10.1.4]

PLAY RECAP *********************************************************************************************
10.10.1.4                  : ok=7    changed=2    unreachable=0    failed=0

```

# Run ansible playbook with dynamic inventory 

If VMs scale in/out dynamically in  my infrastructure, you can't use static inventory. But you can take [dynamic inventory](http://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html) which can pull together all the available VMs in your Azure subscription. I'm filtering the target VMs with tag in this sample. 

I slightly change the [azure_rm.py](https://github.com/ansible/ansible/blob/devel/contrib/inventory/azure_rm.py) to get the private ip address from the 'ansible_host' variable. 

674 line is changed.
```python
host_vars['private_ip_alloc_method'] = ip_config.private_ip_allocation_method
host_vars['ansible_host'] = ip_config.private_ip_address
if ip_config.public_ip_address:
```

### Test anzure_rm.py
You can see all the VM information for 'AnsibleVMGroup' resource group. The most important variable is 'ansible_host'
```
$ ./azure_rm.py --resource-groups=AnsibleVMGroup --pretty
{
  "KoreaSouth": [
    "DatabaseVM",
    "WebVM0",
    "WebVM1"
  ],
  "_meta": {
    "hostvars": {
      "DatabaseVM": {
        "ansible_connection": "winrm",
        "ansible_host": "10.10.1.4",
        "computer_name": "WebVMdatabase",
        "fqdn": null,
        "id": "/subscriptions/e47f0bbb-cd59-41dc-86b7-2e239d536c04/resourceGroups/AnsibleVMGroup/providers/Microsoft.Compute/virtualMachines/DatabaseVM",
        "image": {
          "offer": "WindowsServer",
...
```

### Run webserver playbook with dynamic inventory

```
$ ansible-playbook -i azure_rm.py webserver_dynamic.yml
```

### Run database playbook with dynamic inventory

```
$ ansible-playbook -i azure_rm.py database_dynamic.yml
```
There are 5 filtering options available.
1. location
1. resource group
1. serurity group
1. tag
1. powerstate

Read ["Microsoft Azure Guide"](http://docs.ansible.com/ansible/latest/scenario_guides/guide_azure.html) for more information about azure dynamic inventory. 

# Combine with [Packer](https://www.packer.io/)

You can make a [custom manage image](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/create-vm-generalized-managed) automatically with Packer. If you need to maintain a fresh VM image, you can use Packer and Ansible together. 



