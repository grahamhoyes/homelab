# Ansible Automation

This set of playbooks configures Proxmox, creates a set of VMs, and installs a highly-available k3s cluster with embedded etcd.

## Prerequisites

1. [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) and required modules on the host machine (where you'll be running the playbooks from, likely your workstation). Using pip is the easiest cross-platform way to get the latest version:

    ```bash
    pip install ansible netaddr
    ```

2. [Install the 1password CLI](https://developer.1password.com/docs/cli/get-started/). This is optional, but I use it in [vault-pass.sh](./vault-pass.sh) to fetch the password for ansible-vault.
    - If not using 1password, update `vault-pass.sh` to fetch your password from your secret manager of source and print it to stdout.
    - If you want to use a static password instead, store it in a new file **that you add to .gitignore** and update the `vault_password_section` in [ansible.cfg](ansible.cfg).

3. Upload your public SSH key for the proxmox user in [inventory/group_vars/proxmox.yaml `ansible_user`](inventory/group_vars/proxmox.yaml). You only need to do this for one node in the cluster:


    ```bash
    ssh-copy-id root@192.168.2.30
    ```

4. Configure [resource mapping](https://pve.proxmox.com/wiki/QEMU/KVM_Virtual_Machines#resource_mapping) for host iGPU devices for each node that has an iGPU.

    1. Proxmox Datacenter > Resource Mappigns > PCI Devices > Add
    2. For one node, select the device and give it a name
        - For Intel iGPUs, I go with `intel_igpu` as the name. The PCI device ID is usually `00:02.0`.
    3. A new tree view will appear under PCI Devices. Next to the name you specified, in the Actions column, click the `+` button to add a mapping for each remaining node.
    4. If using a name other than `intel_igpu`, change the variable `proxmox_gpu_mapped_device` in [inventory/group_vars/proxmox.yaml](inventory/group_vars/proxmox.yaml).

## Configuring

### Inventory

The main inventory is in [inventory/inventory.yaml](inventory/inventory.yaml). This has three relevant sections:
- `proxmox_hosts`: The hosts in the proxmox cluster where the k3s VMs will be created
- `k3s_servers`: A "server" node in k3s runs the control plane and embedded etcd database. This section specifies the server VMs to create, including which proxmox node they run on.
    - For high availability, there should be an odd number of entries and at least 3
- `k3s_agents`: An "agent" node is a worker where your workloads actually run. There are some extra options not available for the servers that you can specify:
    - `vlan_id`: If you want the worker to be on a different VLAN for network segmentation.
        - Make sure you configure your network firewall rules so that the server nodes can still reach these VMs
    - `gpu_type`: If a GPU type is specified, a GPU (embedded graphics in my case) will be passed through to the VM. You can only use one of these per proxmox host.

### Group Variables

Various settings and defaults are in [inventory/group_vars](inventory/group_vars/), separated by resource type.

Settings for k3s itself are in [vars/k3s.yaml](vars/k3s.yaml), as these can't be loaded until the VMs are created part way through the playbook.

### Secrets

Secrets (API tokens, VM passwords, etc) are configured using `ansible-vault`. The vault file is in [inventory/group_vars/all/vault.yaml](inventory/group_vars/all/vault.yaml). As above, [vault-pass.sh](vault-pass.sh) fetches the encryption key for the vault from the 1password CLI.

To edit the values, run:

```bash
ansible-vault edit inventory/group_vars/all/vault.yaml
```

This file must contain the following fields:
- `vault_proxmox_api_token_secret`: Value of the proxmox API token
- `vault_proxmox_root_password`: Password for the `root@pam` user. This is currently needed for importing cloud init disk images due to a limitation in the Proxmox API.
- `vault_vm_default_password`: Password for the default user on created VMs. This can be whatever you want.
- `vault_k3s_token`: Token for k3s nodes to communicate with.

To generate the proxmox API tokens from the Proxmox web GUI:
1. Navigate to Datacenter > Permissions > API Tokens
2. Click Add
3. Choose a user with sufficient permissions to create VMs and templates. The default `root@pam` is fine for homelab use.
4. Un-check "Privilege Separation", which will give the API token full root access
    - This is obviously not ideal. Don't do this in a real production setting!

The token ID and associated user need to be set in [inventory/group_vars/all/vars.yaml](inventory/group_vars/all/vars.yaml). The user must include the `@pam` suffix. For example, with a token named `ansible-automation`:

```yaml
# inventory/group_vars/all/vars.yaml
proxmox_api_user: "root@pam"
proxmox_api_token_id: "ansible-automation"
```

## Playbooks

### Configure Proxmox

```bash
ansible-playboook configure-proxmox.yaml
```

This playbook will:
- Install packages needed for subsequent playbooks
- Configure GRUB and add kernel modules to support Intel iGPU passthrough
- Reboot the server

> [!WARNING]
> 🚨 **This will remove the ability to use onboard graphics to access the Proxmox console** 🚨
>
> You won't be able to use monitors connected to the onboard GPU after Proxmox boots.


### Provision VMs and Install k3s

```bash
ansible-playbook site.yaml
```

This playbook will:
- Download an Ubuntu cloud image
- Create Proxmox VMs for k3s control plane and worker nodes
- Install k3s

This playbook uses [Techno Tim's k3s-ansible playbook](https://github.com/techno-tim/k3s-ansible) internally to configure HA k3s with kube-vip and MetalLB.


### Reset VMs

```bash
ansible-playbook reset.yaml
```

This will completely delete the Proxmox VMs and associated disks, good for starting fresh.
