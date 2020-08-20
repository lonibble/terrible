# kvm-deployment

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0) ![Terraform Version](https://img.shields.io/badge/Terraform-v0.12-yellowgreen) ![Ansible Version](https://img.shields.io/badge/Ansible-v2.9%2B-yellowgreen)

This **Ansible** role allow you to initialize and then deploy virtual machines through **Terraform** on a **KVM** server.

## How It Works

The idea comes from the difficulty to automate the deployment of the VMs on a **KVM** environment.

For this reason we decided to automate as much as possible the process of the deployment, using **Ansible** and **Jinja2**.

First of all we provided a basic **HCL** file (templated with **Jinja2**) to describe a basic VM implementation. This is what is usually called IaC.

Then using **Terraform** and its amazing provider for **libvirt** (https://github.com/dmacvicar/terraform-provider-libvirt), we deploy the resultants **HCL** files generated by **Ansible**.

The figure below describe the process in an easy way.

![![kvm-deployment]()](./pics/kvm-deployment.png)

As you can see, we start having the templated file (`terraform-vm.tf.j2`). 

When Ansible run, it generates *n* `.tf` files, depending on the VM specified into the inventory. This is the result of the **init** phase. Once finished this task, the files are completed and ready to be used by **Terraform**.

At this time, Ansible takes these files and use **Terraform** for each instance of them. Once finished this task, the VMs, previously described into the inventory, are correctly deployed into the **KVM** server(s).

## Configuration

First of all you have to compose the inventory file in a right way. This means you have to describe the VMs you want to deploy into the server. 

As you are going to check, there are some interesting variables you are allowed to use to properly describe your infrastructure. Some of them are `required` and some others are `optional`.

Below you can check the basic structure for the inventory.

```yaml
all:
    vars:
        ...
    children:
        hypervisor_1:
            vars:
                provider_uri: ...
                pool_name: ...
               ...
            children:
                group_1:
                    hosts:
                        host_1:
                            ...
                    vars:
                        ...
                group_2:
                    hosts:
                        host_2:
                            ...
                        host_3:
                            ...
        hypervisor_2:
            vars:
                provider_uri: ...
                pool_name: ...
               ...
            children:
                group_3:
                    hosts:
                        host_4:
                            ...
                    vars:
                        ...
                group_4:
                    hosts:
                        host_5:
                            ...
```

Under the 1st `vars` tag, you can specify the various hypervisors you want to use to distribute your infrastructure.

Here's a little example:

```yaml
all:
    vars:
        hypervisor_1:
            vars:
                provider_uri: "qemu:///system"
                pool_name: default
                terraform_node: 127.0.0.1
        ...
```

In this example we specified the *uri* of the KVM server (which is going to be common for all the VMs in this hypervisor group), 
the *storage pool* name of the KVM server and the *terraform node* address, which specify where Terraform is installed and where is going to be ran.

Now, for each VM we want to specify some property such as the number of the cpu(s), memory ram, mac_address, etc.

Here's a little example:

```yaml
        ...
        hypervisor_1:
            vars:
                provider_uri: "qemu:///system"
                pool_name: default
                terraform_node: 127.0.0.1
                disk_source: "~/VirtualMachines/centos8-terraform.qcow2"
            children:
                group_1:
                    hosts:
                        host_1:
                            cpu: 4
                            memory: 8192
                            network_name: "default"
                            mac_address: "de:ad:be:ef:00:11"
                group_2:
                    hosts:
                        host_2:
                            cpu: 2
                        host_3:
                            disk_source: "~/VirtualMachines/opensuse15.2-terraform.qcow2"
                            cpu: 4
                            memory: 4096
```

In this example, we specified 2 main groups (`group_1`, `group_2`) inside the `hypervisor_1`.
These groups are composed overall by 3 machines (`host_1`, `host_2`, `host_3`). 
As you can see, not all the properties are specified for each machine. This is due to the default values of the variables provided by this role. 

Thanks to the **variabe hierarchy** in Ansible, you can configure variables:

- Hypervisor wise
- VM groups wise
- Single VM wise

This will make easier to manage large homogeneous clusters, still retaining the power of per-VM customization.

![Workflow](pics/workflow.png)

In the example above, we can see, for `hypervisor_1`, the default OS for VMs is **Centos**, but we specified for `host_3` to be an **OpenSuse** node. Similarly for the `hypervisor_2`, the default OS for VMs is **Ubuntu**, but we specified  for the `host_4` to be a **Centos** node.

This is valid for **any** variable in the playbook.

You can check the default values under `default/main.yml`.

### Variables

Once you understand how to fill the inventory file, you are ready to check all the available variables to generate your infrastructure.

These variables are **required**, they should be declared on per-hypervisor scope:

* **ansible_host:** `required`. Specify the ip address for the VM. If not specified, a random ip is assigned.
* **bastion_enabled:** `required`. Specify `True` or `False`. In case the **terraform_node** and KVM server differ, you should enable the bastion. This will enable the use of the KVM server as jumphost to enter th VMs via ssh.
* **bastion_host:** `required` if **bastion_enabled** is `True`. Specify the ip address of the bastion host.
* **bastion_password:** `required` if **bastion_enabled** is `True`. Specify the password of the bastion host.
* **bastion_port:** `required` if **bastion_enabled** is `True`. Specify the port (for the ssh connection) of the bastion host.
* **bastion_user:** `required` if **bastion_enabled** is `True`. Specify the user of the bastion host.
* **disk_source:** `required`. Specify the (local) path to the virtual disk you want to use to deploy the VMs.
* **pool_name:** `required`. Specify the *storage pool* name where you want to deploy the VMs on the KVM server.
* **provider_uri:** `required`. Specify the uri of the KVM server. You can use `qemu:///system` if the KVM server is you own computer, or `qemu+ssh://USER@IP:PORT/system` (or different) if your server is remote.
* **ssh_password:** `required`. Specify the password to access the deployed VMs.
* **ssh_port:** `required`. Specify the port to access the deployed VMs.
* **ssh_user:** `required`. Specify the user to access the deployed VMs.
* **terraform_node:** `required`. Specify the ip of the machine that performs the Terraform tasks. The default value of 127.0.0.1 indicates that the machine that perform Terraform tasks is the same that launches the Ansible playbook. In case the Terraform machine is not the local machine, put the ip/hostname of the Terraform node.

These variable are optional, there are sensible defaults set up, most of them can be declared from **hypervisor scope** to **vm-group scope** and **per-vm scope**:

* **ansible_ssh_pass:** `optional`. Specify a new password to access the Vm. If not specified, the default value (**ssh_password**) is taken.
* **cpu:** `optional`. Specify the cpu number for the VM. If not specified, the default value is taken. Default: `1`
* **mac_address:** `optional`. Specify the memory ram for the VM. If not specified, a random mac is assigned.
* **memory:** `optional`. Specify the memory ram for the VM. If not specified, the default value is taken. Default: `1024`
* **network_name:** `optional`. Specify the network name for the VM. If not specified, the default value is taken. Default: `"default"`
* **change_passwd_cmd:** `optional`. Specify a different command to be used to change the password to the user. If not specified the default command is used. Default: `echo root:{{ ansible_ssh_pass }} | chpasswd`. This variable become really useful when you are using a FreeBSD OS.

## Support

Actually the role is supporting the most common 3 OS families for the Guests:

* RedHat
* Debian
* Suse
* FreeBSD

This means you are able to generate the infrastructure using the OS listed above.

Hypervisor OS is agnostic, as long as **libvirt** is installed.

## Usage

Once composed the inventory file, it's time to run your playbook.

```bash
ansible-playbook -i inventory.yml -u root main.yml
```
