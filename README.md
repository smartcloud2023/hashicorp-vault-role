# HashiCorp Vault on AWS with Ansible

This repo builds a small, production-style HashiCorp Vault setup on AWS using Ansible roles. Vault itself only talks over private IPs inside the VPC. The one public entry point is HAProxy, which serves **https://group3assessment.online** with a Let's Encrypt certificate.

What you end up with:

- **3 Vault servers** running Raft integrated storage (3 voters, one leader)
- **1 transit server** that auto-unseals the Vault servers, so nobody has to type unseal keys after a reboot
- **HAProxy** in front, with Let's Encrypt on the public side and our own internal CA on the private side
- **TLS everywhere**: users to HAProxy, HAProxy to Vault, Vault to Vault, Vault to transit
- **No secrets on the Ansible machine**: unseal keys, recovery keys and tokens live in AWS Secrets Manager

---

## Architecture

![Vault architecture](docs/vault-architecture.png)

- Diagram on Google Drive: [vault-architecture.drawio](https://drive.google.com/file/d/1-db2WA_KoyqpP7v6e36-6kU5Sl9r_tAr/view?usp=sharing)
- Editable copy in this repo: [`docs/vault-architecture.drawio`](docs/vault-architecture.drawio). Open it at [app.diagrams.net](https://app.diagrams.net).

> To make the picture above show up, open the `.drawio` file in draw.io, go to **File → Export as → PNG**, and save it as `docs/vault-architecture.png`.

How traffic flows:

1. A user opens `https://group3assessment.online`. DNS points to the public IP of HAProxy.
2. HAProxy terminates the Let's Encrypt TLS, then opens a new TLS connection to the active Vault node on its private IP. It checks the Vault certificate against our internal CA.
3. The three Vault nodes replicate data to each other with Raft over TLS on port 8201.
4. When a Vault node starts, it asks the transit server to decrypt its unseal key. This is auto-unseal.
5. Ansible runs from Master-Node over SSH. When it needs a key or token, it reads it from AWS Secrets Manager using the instance's IAM role.

---

## Servers

| Name | Role | Private IP | Notes |
|---|---|---|---|
| lb-1 | HAProxy + Let's Encrypt | 172.31.31.224 | Public IP 18.145.123.49, the only server open to the internet |
| vault-1 | Vault (voter) | 172.31.16.6 | Usually the leader |
| vault-2 | Vault (voter) | 172.31.25.70 | |
| vault-3 | Vault (voter) | 172.31.19.65 | |
| transit-1 | Vault transit (auto-unseal) | 172.31.20.5 | |
| Master-Node | Ansible control node | same VPC | Runs the playbooks, stores no secrets |

All servers run Ubuntu 22.04 or 24.04 and log in as `ubuntu`.

Suggested sizes for a lab or small production setup: 2 vCPU / 4 GB RAM / 50 GB SSD for each Vault node, and 1 vCPU / 1–2 GB for transit and HAProxy.

---

## Repo layout

```
vault-roles/
├── ansible.cfg                  # inventory path, roles path, SSH settings
├── inventory.ini                # the 5 servers
├── group_vars/all.yml           # domain, email, Vault version, AWS region
├── site.yml                     # runs the roles in order (each step has a tag)
├── requirements.yml             # Ansible collections for AWS
├── files/                       # Vault zip is downloaded here automatically
├── docs/
│   ├── vault-architecture.drawio
│   └── iam-policy-master-node.json
└── roles/
    ├── certs/          # Step 1: internal CA + server certificates
    ├── aws_secrets/    # One-off: move old local secrets into Secrets Manager
    ├── vault_install/  # Step 2: Vault binary, user, folders, certs, systemd
    ├── transit/        # Step 3: transit server, unseal key, policy, token
    ├── vault_cluster/  # Step 4: 3 Vault voters, init, auto-unseal
    ├── haproxy/        # Step 5: HAProxy + Let's Encrypt
    └── check/          # Step 6: confirm the cluster through the public URL
```

---

## Before you start

### 1. AWS

**Security groups.** These are the only rules you need:

| Server | Port | From |
|---|---|---|
| lb-1 | 80, 443 | `0.0.0.0/0` (Let's Encrypt needs port 80) |
| vault-1/2/3 | 8200, 8201 | the VPC (`172.31.0.0/16`) |
| transit-1 | 8200 | the VPC |
| all servers | 22 | Master-Node's private IP |

**IAM role for Master-Node.**

1. Create a policy from [`docs/iam-policy-master-node.json`](docs/iam-policy-master-node.json). It only allows access to secrets that start with `hashicorp-vault/`.
2. Create an IAM role for EC2 and attach that policy. We called it `vault-role-secret`.
3. Attach the role to Master-Node: **EC2 → Master-Node → Actions → Security → Modify IAM role**.

### 2. DNS

Point `group3assessment.online` (an A record) to the **public** IP of lb-1, then check it:

```bash
dig +short group3assessment.online
# should print 18.145.123.49
```

### 3. Master-Node

Install Ansible, the AWS libraries and the collections:

```bash
sudo apt update
sudo apt install -y ansible python3-boto3 openssl
ansible-galaxy collection install -r requirements.yml
```

Put your SSH key in place:

```bash
cp sonarkey1.pem ~/sonarkey1.pem
chmod 600 ~/sonarkey1.pem
```

Copy this repo to Master-Node (for example `~/hashicorp-vault-role`) and `cd` into it. Run all the commands below from inside that folder, as the `ubuntu` user. Don't use `sudo` with `ansible-playbook`; Ansible already uses sudo on the servers where it needs to.

---

## Configure

### inventory.ini

Use **private IPs** for everything. Master-Node is in the same VPC, so it can reach them directly.

```ini
[transit]
transit-1 ansible_host=172.31.20.5

[vault]
vault-1 ansible_host=172.31.16.6
vault-2 ansible_host=172.31.25.70
vault-3 ansible_host=172.31.19.65

[lb]
lb-1 ansible_host=172.31.31.224 public_ip=18.145.123.49

[servers:children]
transit
vault
lb

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/sonarkey1.pem

[servers:vars]
ansible_become=true
```

`ansible_become=true` is set on `servers` only. If you put it under `all`, Ansible also runs the local certificate step with sudo, and you end up with root-owned files you can't read.

### group_vars/all.yml

The values you'll usually change:

```yaml
vault_version: "1.20.4"
public_domain: group3assessment.online
letsencrypt_email: you@yourdomain.com     # must be a real address
aws_region: us-west-1
aws_secret_prefix: hashicorp-vault
```

### Check that Ansible can reach every server

```bash
ansible all -m ping
```

All five servers should answer with `pong`. If one doesn't, fix that first (see [Troubleshooting](#troubleshooting)).

---

## Deploy everything at once

```bash
ansible-playbook site.yml
```

That runs all six steps in order. The first run takes about 5–10 minutes, most of it package installs and waiting for the cluster to form.

It's safe to run again. Steps that are already done are skipped. For example, Vault won't be re-initialised and a new Let's Encrypt certificate won't be requested.

---

## Deploy step by step (using tags)

Running one step at a time makes it much easier to see where something went wrong. Each step has its own tag:

| Step | Tag | Runs on | What it does |
|---|---|---|---|
| 1 | `certs` | Master-Node | Creates the internal CA and a TLS certificate for each Vault server |
| 2 | `install` | transit-1, vault-1/2/3 | Installs Vault, copies the certificates, installs the systemd service |
| 3 | `transit` | transit-1 | Starts and initialises transit, unseals it, creates the auto-unseal key and token |
| 4 | `vault` | vault-1/2/3 | Starts the 3 voters, initialises the cluster, waits until all are unsealed |
| 5 | `lb` | lb-1 | Gets the Let's Encrypt certificate and starts HAProxy |
| 6 | `check` | Master-Node | Lists the cluster through the public URL |

Run them in this order:

```bash
ansible-playbook site.yml --tags certs
ansible-playbook site.yml --tags install
ansible-playbook site.yml --tags transit
ansible-playbook site.yml --tags vault
ansible-playbook site.yml --tags lb
ansible-playbook site.yml --tags check
```

You can also combine tags, for example `--tags certs,install`. To see every task without running anything: `ansible-playbook site.yml --list-tasks`.

### What to check after each step

**After `certs`**

```bash
openssl verify -x509_strict -CAfile pki/ca.crt pki/transit-1.crt pki/vault-1.crt pki/vault-2.crt pki/vault-3.crt
```

All four should say `OK`.

**After `install`**

```bash
ansible transit,vault -m command -a "/usr/local/bin/vault version"
```

**After `transit`**

```bash
ansible transit -b -m command -a "vault status -address=https://172.31.20.5:8200 -ca-cert=/etc/vault.d/tls/ca.crt"
```

Look for `Initialized true` and `Sealed false`.

**After `vault`**

```bash
ansible vault -b -m shell -a "vault status -address=https://{{ ansible_host }}:8200 -ca-cert=/etc/vault.d/tls/ca.crt | grep -E 'Seal Type|Sealed|HA Mode'"
```

Every node should show `Seal Type transit` and `Sealed false`. One node is `active`, the other two are `standby`.

**After `lb`**

```bash
curl -s https://group3assessment.online/v1/sys/health | python3 -m json.tool
```

Look for `"initialized": true` and `"sealed": false`, with no certificate warning.

**After `check`**

The output lists vault-1, vault-2 and vault-3, with one leader and all three as voters.

---

## Secrets

When Vault and transit are initialised, they hand out unseal keys, recovery keys and root tokens. We don't keep any of these on Master-Node. They go straight into AWS Secrets Manager:

| Secret name | What's in it |
|---|---|
| `hashicorp-vault/transit-keys` | Transit unseal keys (3, any 2 unseal it) and the transit root token |
| `hashicorp-vault/vault-keys` | Vault recovery keys (3, any 2 needed) and the Vault root token |
| `hashicorp-vault/autounseal-token` | The token the Vault nodes use to reach transit |

The auto-unseal token also has to live on the 3 Vault nodes, in `/etc/vault.d/vault.env` (mode 0600). They need it to unseal.

### Moving old local secrets to AWS (one time only)

If you ran an older version of this project, your keys may still be in `secrets/` on Master-Node. Move them:

```bash
# copy to AWS and confirm each one ("stored in AWS: True")
ansible-playbook site.yml --tags aws-secrets

# make sure the playbook can read them back
ansible-playbook site.yml --tags check

# delete the local copies (only the ones confirmed in AWS)
ansible-playbook site.yml --tags aws-secrets -e aws_secrets_delete_local=true
```

The `aws-secrets` step only runs when you ask for it by tag. Running `ansible-playbook site.yml` on its own skips it.

---

## Logging in to Vault

Get the root token from Secrets Manager. Either use the AWS Console (**Secrets Manager → `hashicorp-vault/vault-keys` → Retrieve secret value → `root_token`**), or run this on Master-Node:

```bash
aws secretsmanager get-secret-value \
  --secret-id hashicorp-vault/vault-keys --region us-west-1 \
  --query SecretString --output text \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["root_token"])'
```

Then open **https://group3assessment.online**, choose **Method: Token**, and paste it in.

Use the root token only for the initial setup. Create a proper admin login (userpass or OIDC with MFA), then revoke the root token. You can always generate a new one later with the recovery keys.

---

## Day-to-day operations

| Task | How |
|---|---|
| Re-apply config after a change | `ansible-playbook site.yml`. It only changes what's different. |
| A Vault node reboots | Nothing to do. It unseals itself through transit. |
| transit-1 reboots | Run `ansible-playbook site.yml --tags transit`. It unseals transit using the keys in Secrets Manager. Vault nodes that restarted while transit was down will then unseal themselves. |
| Let's Encrypt renewal | Automatic. Certbot's timer runs twice a day and reloads HAProxy. |
| Internal certificates (valid 1 year) | Delete the server `.crt` files in `pki/` and run `--tags certs,install`. Vault restarts with the new certificates. |
| Upgrade Vault | Change `vault_version` in `group_vars/all.yml`, delete `/usr/local/bin/vault` on the nodes, run `--tags install`, then restart the nodes one at a time. |

---

## Troubleshooting

These are the problems we actually hit while building this, and how we fixed them.

| Error | Cause | Fix |
|---|---|---|
| `Host key verification failed` | Master-Node has never connected to the server before | `ansible.cfg` already uses `StrictHostKeyChecking=accept-new`. Otherwise run `ssh-keyscan` for each IP. |
| `Could not find or access '.../pki/ca.crt'` | `pki/` is owned by root, because `ansible_become` was set for `all` or the playbook was run with sudo | `sudo chown -R ubuntu:ubuntu ~/hashicorp-vault-role` and keep `ansible_become` under `[servers:vars]` |
| `CA cert does not include key usage extension` | Python 3.13+ rejects CA certificates without key usage | Already fixed in the `certs` role. Old certificates are detected and recreated automatically. |
| `AccessDeniedException ... GetResourcePolicy` | The IAM policy is missing a permission | Use the full policy in `docs/iam-policy-master-node.json` and wait a minute for IAM to apply it |
| `Unable to locate credentials` | Master-Node has no IAM role | Attach `vault-role-secret` to the instance |
| Let's Encrypt fails | DNS doesn't point to lb-1 yet, or port 80 is closed | Check `dig +short group3assessment.online` and the security group |
| `Wait until every node has joined` keeps retrying | Port 8201 is blocked between the Vault nodes, or transit is sealed | Open 8200/8201 inside the VPC, then run `--tags transit` |

Useful logs on the servers:

```bash
sudo journalctl -u vault -f        # Vault
sudo journalctl -u haproxy -f      # HAProxy
sudo journalctl -u certbot -f      # Let's Encrypt renewals
```

---

## Security notes

- Only HAProxy is reachable from the internet. Vault and transit listen on private IPs.
- Every connection uses TLS. Internally we use our own CA. Publicly we use Let's Encrypt.
- Secrets sit in AWS Secrets Manager behind an IAM role that only sees `hashicorp-vault/*`. Every read shows up in CloudTrail.
- The transit token can only encrypt and decrypt with one key. It can't read anything else.

Still on the to-do list:

1. Create an admin login with MFA and revoke both root tokens.
2. Turn on the Vault audit log and send it to CloudWatch.
3. Move `pki/ca.key` and the server private keys off Master-Node, to Secrets Manager or Vault PKI.
4. Replace the transit server with AWS KMS auto-unseal, which means one less server and no unseal keys at all.
