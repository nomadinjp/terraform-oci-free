# OCI Free Tier Ansible Deployment

Deploy Terraform-based OCI Free Tier instance renewal with Ansible.

## Features

- Terraform runs in Docker container
- Cron-based scheduling:
  - **Daytime (06:00-22:00)**: Every 2 minutes
  - **Nighttime (22:00-06:00)**: Every 1 minute
- Telegram notification on successful instance creation
- Lock file prevents concurrent execution
- Automatic log rotation
- **Stops running after successful instance creation** (saves resources)

## How It Works

1. Cron runs the script repeatedly
2. Script tries to create an instance with Terraform
3. Once successful, a flag file is created
4. Subsequent runs see the flag and skip Terraform (just log and exit)
5. When instance is lost, run `reset.yml` to clear the flag and restart

## Prerequisites

1. **Target Server**: Linux server with Docker installed
2. **Ansible Control Machine**: Ansible installed
3. **Telegram Bot**: Create bot via [@BotFather](https://t.me/BotFather)
4. **Get Chat ID**: Message [@userinfobot](https://t.me/userinfobot) on Telegram

> **Note**: If Docker is not installed on the target server, run `ansible-playbook docker-setup.yml` first.

## Setup

### 1. Copy Example Files

```bash
cd ansible
cp inventory.ini.example inventory.ini
cp vars/secrets.yml.example vars/secrets.yml
```

### 2. Configure Inventory

Edit `ansible/inventory.ini`:

```ini
[oci_free_servers]
oci-server ansible_host=YOUR_SERVER_IP ansible_user=root
```

### 3. Configure Secrets

Edit `ansible/vars/secrets.yml`:

```yaml
# OCI Credentials (from terraform.tfvars)
oci_user: "ocid1.user.oc1..."
oci_fingerprint: "xx:xx:xx..."
oci_tenancy: "ocid1.tenancy.oc1..."
oci_region: "ap-tokyo-1"
oci_namespace: "your-namespace"

# Telegram
telegram_bot_token: "123456:ABC-DEF..."
telegram_chat_id: "123456789"
```

### 4. Prepare Keys on Target Server

Before running the playbook, copy your keys to the target server:

```bash
# OCI API key
scp ~/.oci/oci.pem root@YOUR_SERVER:/root/.oci/

# SSH public key for instance
scp ~/.ssh/oci_free.pub root@YOUR_SERVER:/root/.ssh/
```

### 5. Deploy

```bash
cd ansible
ansible-playbook -i inventory.ini deploy.yml
```

## After Deployment

### View Logs

```bash
# Terraform output
tail -f /opt/oci-free/logs/terraform.log

# Cron output
tail -f /opt/oci-free/logs/cron.log
```

### Test Manually

```bash
ssh root@YOUR_SERVER
/opt/oci-free/run.sh
```

### Check Cron Jobs

```bash
crontab -l
```

### Remove Deployment (Cleanup)

To completely remove the deployment:

```bash
cd ansible
ansible-playbook -i inventory.ini cleanup.yml
```

This removes:
- Cron jobs
- Working directory `/opt/oci-free`
- Logrotate config

### Reset / Restart Instance Creation

When the instance is lost and you want to start trying again:

```bash
cd ansible
ansible-playbook -i inventory.ini reset.yml
```

This removes the success flag, allowing cron to resume instance creation attempts.

### Manual Reset on Server

```bash
ssh root@YOUR_SERVER
rm -f /opt/oci-free/lock/.instance_created
```

## Directory Structure on Target

```
/opt/oci-free/
├── terraform/           # Terraform files
├── run.sh              # Main script
├── logs/
│   ├── terraform.log   # Terraform output
│   └── cron.log        # Cron output
└── lock/
    └── terraform.lock  # Lock file
```

## Troubleshooting

### Lock File Stuck

```bash
rm -f /opt/oci-free/lock/terraform.lock
```

### Docker Container Issues

```bash
# Check Docker is running
systemctl status docker

# Test Terraform container
docker run --rm hashicorp/terraform:1.9 version
```

### Telegram Not Working

```bash
# Test Telegram API
curl -X POST "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d "chat_id=<CHAT_ID>" \
  -d "text=Test message"
```

## Timezone

Server timezone is set to `Asia/Tokyo` for consistent cron scheduling.
