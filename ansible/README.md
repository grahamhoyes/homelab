# Ansible Automation

This set of playbooks configures Proxmox, creates a set of VMs, and installs a highly-available k3s cluster with embedded etcd.

## Prerequisites

1. [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) on the host machine (where you'll be running the playbooks from, likely your workstation). Using pip is the easiest cross-platform way to get the latest version:

    ```bash
    pip install ansible
    ```

2. [Install the 1password CLI](https://developer.1password.com/docs/cli/get-started/). This is optional, but I use it in [vault-pass.sh](./vault-pass.sh) to fetch the password for ansible-vault.
    - If not using 1password, update `vault-pass.sh` to fetch your password from your secret manager of source and print it to stdout.
    - If you want to use a static password instead, store it in a new file **that you add to .gitignore** and update the `vault_password_section` in [ansible.cfg](ansible.cfg).

3. Upload your public SSH key for the proxmox user in [inventory/group_vars/proxmox.yaml `ansible_user`](inventory/group_vars/proxmox.yaml). You only need to do this for one node in the cluster:


    ```bash
    ssh-copy-id root@192.168.2.30
    ```

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

### Secrets

Secrets (API tokens, VM passwords, etc) are configured using `ansible-vault`. The vault file is in [inventory/group_vars/all/vault.yaml](inventory/group_vars/all/vault.yaml). As above, [vault-pass.sh](vault-pass.sh) fetches the encryption key for the vault from the 1password CLI.

To edit the values, run:

```bash
ansible-vault edit inventory/group_vars/all/vault.yaml
```

This file must contain the following fields:
- `vault_proxmox_api_token_secret` Value of the proxmox API token
- `vault_vm_default_password`: Password for the default user on created VMs. This can be whatever you want.

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


### Provision VMs

```bash
ansible-playbook vm-provision.yaml
```