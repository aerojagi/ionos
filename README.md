# IONOS Cloud Ansible Lab

Infrastructure automation examples for **IONOS Cloud** using **Ansible**.

This repository provides a small but practical lab workflow that covers:

- identity management in IONOS Cloud
- user and group administration
- VDC and VM provisioning
- SSH access configuration
- dynamic inventory generation

It is designed as a simple foundation for learning, testing, and extending IONOS Cloud automation with Ansible.

---

## Features

- Creates an **administrator user**
- Creates **standard users** and **groups**
- Provisions a **small Ubuntu VM** in a **VDC**
- Injects your **SSH public key**
- Configures **NIC firewall rules** for SSH access
- Generates a **dynamic inventory file** from the deployed environment

---

## Playbooks

| Playbook | Purpose |
|---|---|
| `playbooks/01-user-admin.yml` | Ensure an IONOS administrator user exists |
| `playbooks/02-users-groups.yml` | Ensure non-admin users and groups exist, and assign users to groups |
| `playbooks/03-create-vdc-vm.yml` | Create a VDC, LAN, Ubuntu VM, SSH access, and firewall rule |
| `playbooks/04-dynamic-inventory.yml` | Query IONOS Cloud and generate `inventory_ionos.yml` |

---

## Repository layout

```text
.
├── ansible.cfg
├── inventory
├── group_vars/
│   └── local/
│       └── vault.yml
├── ionos.token
├── playbooks/
│   ├── 01-user-admin.yml
│   ├── 02-users-groups.yml
│   ├── 03-create-vdc-vm.yml
│   └── 04-dynamic-inventory.yml
└── README.md
```

---

## Requirements

Make sure the following are available on the controller host:

- Ansible
- `ionoscloudsdk.ionoscloud` Ansible collection
- Valid IONOS Cloud credentials
- Ansible Vault password
- SSH key pair on the controller

Example checks:

```bash
ansible --version
ansible-galaxy collection list | grep ionoscloudsdk.ionoscloud
ls -l ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

---

## Authentication

This lab supports the following authentication methods:

- **IONOS API token**
- **IONOS username and password**

Recommended: use an **API token**.

### Option 1: export token in the shell

```bash
export IONOS_TOKEN="$(tr -d '\n' < ionos.token)"
```

### Option 2: store credentials in Ansible Vault

Example `group_vars/local/vault.yml`:

```yaml
ionos_token: "REDACTED"
ionos_ansible_admin_password: "REDACTED"
ionos_default_user_password: "REDACTED"
```

Run the playbooks with:

```bash
ansible-playbook -i inventory <playbook> --ask-vault-pass
```

---

## Inventory setup

The playbooks are intended to run locally from the controller host.

Example `inventory`:

```ini
[local]
localhost ansible_connection=local
```

This is important because Ansible automatically loads variables from:

```text
group_vars/local/
```

The inventory group name must therefore match `local`.

---

## Variables

### Required vault variables

The following variables are commonly required:

- `ionos_token`
- `ionos_ansible_admin_password`
- `ionos_default_user_password`

### Runtime variables

Some values are passed at execution time, for example:

- `admin_email`
- `admin_firstname`
- `admin_lastname`

---

## Quick start

Run the playbooks in the following order.

### 1. Ensure the administrator user exists

```bash
ansible-playbook -i inventory playbooks/01-user-admin.yml \
  -e "admin_email=admin@example.com admin_firstname=Ansible admin_lastname=Admin" \
  --ask-vault-pass
```

This creates the IONOS **administrator user** if it does not already exist.

---

### 2. Ensure standard users and groups exist

```bash
ansible-playbook -i inventory playbooks/02-users-groups.yml --ask-vault-pass
```

This playbook:

- creates non-admin users
- creates the configured group
- adds users to the group

For consistency, group membership should be managed from **one side only**. In this repository, membership is managed from the **group side**.

---

### 3. Create the VDC and small Ubuntu VM

```bash
ansible-playbook -i inventory playbooks/03-create-vdc-vm.yml --ask-vault-pass
```

This playbook:

- creates a VDC
- creates a public LAN
- creates a small Ubuntu VM
- injects the local SSH public key
- enables the NIC firewall
- opens TCP port 22 for SSH

Typical lab sizing:

- **1 shared vCPU**
- **2048 MB RAM**
- **20 GB disk**
- **Ubuntu public image**
- **public IPv4**
- **SSH key authentication**

After deployment, connect with:

```bash
ssh root@<public_ip>
```

If needed:

```bash
ssh -i ~/.ssh/id_ed25519 root@<public_ip>
```

---

### 4. Generate inventory from IONOS Cloud

```bash
ansible-playbook -i inventory playbooks/04-dynamic-inventory.yml --ask-vault-pass
```

This generates:

```text
inventory_ionos.yml
```

Validate it with:

```bash
cat inventory_ionos.yml
ansible-inventory -i inventory_ionos.yml --graph
ansible ionos_vdc_hosts -i inventory_ionos.yml -m ping
```

---

## SSH key usage

The VM creation playbook expects the following public key by default:

```text
~/.ssh/id_ed25519.pub
```

Check it with:

```bash
cat ~/.ssh/id_ed25519.pub
```

The matching private key is later used to access the VM:

```text
~/.ssh/id_ed25519
```

Do not commit private keys to the repository.

---

## Idempotency

These playbooks are written to be idempotent or close to idempotent:

- `01-user-admin.yml` ensures the admin user exists
- `02-users-groups.yml` ensures users, groups, and memberships exist
- `03-create-vdc-vm.yml` ensures infrastructure exists
- `04-dynamic-inventory.yml` regenerates inventory from current cloud state

Recommended practice:

- run the same playbook more than once
- verify the second run reports minimal or no changes
- use `--check --diff` where appropriate

Example:

```bash
ansible-playbook -i inventory playbooks/01-user-admin.yml \
  -e "admin_email=admin@example.com admin_firstname=Ansible admin_lastname=Admin" \
  --ask-vault-pass --check --diff
```

---

## Security considerations

- Never commit real credentials to Git
- Store secrets in **Ansible Vault**
- Restrict SSH access to your own public IP whenever possible
- Avoid exposing SSH to `0.0.0.0/0` in long-lived environments
- Do not paste private keys into playbooks or variable files
- Pin exact OS image versions for reproducible builds

---

## Troubleshooting

### Vault decryption problems

If you see:

```text
Attempting to decrypt but no vault secrets found
```

or:

```text
Decryption failed
```

check the following:

- the vault password was provided
- the vault password is correct
- the inventory group matches the `group_vars` path

Useful commands:

```bash
grep -R "ANSIBLE_VAULT" .
cat inventory
find group_vars -type f
```

---

### Missing IONOS credentials

If you see:

```text
Token or username & password are required
```

then Ansible did not receive valid IONOS authentication.

Check:

- `ionos_token` exists in the vault file
- `IONOS_TOKEN` is exported in the current shell
- username and password variables are defined correctly

---

### SSH access fails

If SSH connects to the host but returns:

```text
Permission denied (publickey)
```

check:

- the correct SSH user is used
- the matching private key exists locally
- the public key was injected during VM creation
- port 22 is allowed by the NIC firewall

For the Ubuntu image used in this lab, connect as:

```bash
ssh root@<public_ip>
```

---

### Inventory generation fails

Run with more verbosity:

```bash
ansible-playbook -i inventory playbooks/04-dynamic-inventory.yml --ask-vault-pass -vvv
```

Then verify:

- the VDC exists
- at least one server exists in the VDC
- the NIC has a public IP

---

## Example workflow

```bash
export IONOS_TOKEN="$(tr -d '\n' < ionos.token)"

ansible-playbook -i inventory playbooks/01-user-admin.yml \
  -e "admin_email=admin@example.com admin_firstname=Ansible admin_lastname=Admin" \
  --ask-vault-pass

ansible-playbook -i inventory playbooks/02-users-groups.yml --ask-vault-pass

ansible-playbook -i inventory playbooks/03-create-vdc-vm.yml --ask-vault-pass

ansible-playbook -i inventory playbooks/04-dynamic-inventory.yml --ask-vault-pass

ansible ionos_vdc_hosts -i inventory_ionos.yml -m ping
```

---

## Suggested next steps

This lab can be extended further by:

- converting the playbooks into reusable roles
- separating defaults from secrets more clearly
- adding cloud-init for package bootstrapping
- creating private/internal servers
- generating SSH config automatically
- pinning exact Ubuntu image versions
- adding tags for selective execution
- integrating into CI pipelines

---

## Purpose

This repository is intended as a practical learning and lab environment for:

- user and access management
- group-based administration
- cloud infrastructure provisioning
- SSH-based server access
- inventory discovery and automation targeting

It provides a strong starting point for more advanced IONOS Cloud automation projects.
