# Simple Vault cluster with Ansible roles (Ubuntu)

```
 Users ──HTTPS (Let's Encrypt)──▶ HAProxy (lb-1) ──TLS (self-signed)──▶ vault-1 / vault-2 / vault-3  (3 voters)
                                                                             │
                                                                             └── TLS (self-signed) ──▶ transit-1 (auto-unseal)
```

| Connection | Certificate |
|---|---|
| Users → HAProxy (`https://group3assessment.online`) | Let's Encrypt, renewed automatically |
| HAProxy → Vault nodes | Self-signed internal CA (HAProxy verifies it) |
| Vault ↔ Vault, Vault → transit | Self-signed internal CA |

## Layout

```
vault-roles/
├── ansible.cfg
├── inventory.ini            # 5 servers (public + private IPs), user ubuntu, SSH key
├── group_vars/all.yml       # domain, Let's Encrypt email, Vault zip, key counts
├── site.yml                 # calls the 6 roles in order
├── files/                   # put the Vault zip here
└── roles/
    ├── certs/               # Step 1: self-signed CA + server certificates (your machine)
    ├── vault_install/       # Step 2: Vault binary, user, folders, certs, systemd
    ├── transit/             # Step 3: transit node, unseal key, policy, token
    ├── vault_cluster/       # Step 4: 3 voters, init, auto-unseal
    ├── haproxy/             # Step 5: Let's Encrypt + HAProxy
    └── check/               # Step 6: show voters and leader
```

## Roles

| Role | Runs on | What it does |
|---|---|---|
| `certs` | Your machine | Creates the self-signed CA and a certificate for each Vault server |
| `vault_install` | transit + 3 Vault nodes | Installs Vault and copies the certificates |
| `transit` | transit-1 | Starts, initialises and unseals transit; creates the unseal key and a token |
| `vault_cluster` | 3 Vault nodes | Starts all 3, initialises the cluster, the others join and auto-unseal |
| `haproxy` | lb-1 | Gets the Let's Encrypt certificate for the public URL and starts HAProxy |
| `check` | Your machine | Shows the 3 voters and the leader through the public URL |

## Before you run

1. **SSH key:** set `ansible_ssh_private_key_file` in `inventory.ini` and run `chmod 600` on the key.
2. **IPs:** in `inventory.ini` set `ansible_host` (public IP for SSH) and `private_ip` (AWS private IP, used for all Vault traffic) for each server.
3. **Domain:** set `public_domain` and `letsencrypt_email` in `group_vars/all.yml`. Point the domain's DNS A record at the HAProxy server's public IP.
4. **Vault zip:** set `vault_version` in `group_vars/all.yml`. The zip is downloaded into `files/` automatically.
5. **Ports** (security group or firewall):

| Server | Open to | Ports |
|---|---|---|
| lb-1 | The internet | 80 (Let's Encrypt check), 443 (Vault URL) |
| vault-1/2/3 | lb-1 and each other | 8200, 8201 |
| transit-1 | vault-1/2/3 | 8200 |

## Run

```bash
ansible-playbook site.yml
```

Then open `https://group3assessment.online` (your domain).

## After it runs

- `secrets/transit-keys.json`: transit unseal keys and root token
- `secrets/vault-keys.json`: Vault recovery keys and root token
- `secrets/autounseal-token`: token the Vault nodes use to reach transit

Move these somewhere safe. Anyone with them controls Vault.

**If transit-1 reboots:** run `ansible-playbook site.yml` again. It unseals transit, and the 3 Vault nodes unseal themselves.

**Certificates:** Let's Encrypt renews itself through certbot's timer. The internal certificates last 1 year: delete the server `.crt` files in `pki/` and rerun the playbook to renew them.

Written for Ubuntu 22.04 / 24.04.
