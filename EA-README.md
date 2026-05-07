# ea-agent

OpenClaw on AWS EC2 walkthrough.

**Docs:** [OpenClaw](https://docs.openclaw.ai/) | [Model providers](https://docs.openclaw.ai/concepts/model-providers)

---

## Order

| Step | File | Summary |
|------|------|---------|
| 1 | [01-AWS-EC2.md](./walkthrough/01-AWS-EC2.md) | EC2 connection details (existing instance) and how to provision a new one |
| 2 | [02-OPENCLAW-DOCKER.md](./walkthrough/02-OPENCLAW-DOCKER.md) | SSH, Docker, clone repo, build image, run gateway |
| 3 | [03-VERTEX-GEMINI.md](./walkthrough/03-VERTEX-GEMINI.md) | Gemini API key (default) or Vertex via service-account JSON |

---

## Quick recap

1. **Local:** SSH to the EC2 using `keys/open-claw.pem` (host in `keys/help.md`).
2. **On EC2:** Install Docker → clone OpenClaw → create `.env` + `docker-compose.override.yml` + Dockerfile edits → build & run.
3. **Credentials:** Gemini API key in `.env`, or upload `keys/ea-agent-sa.json` to the VM for Vertex.
4. **Access:** SSH tunnel `-L 18789:127.0.0.1:18789` → http://127.0.0.1:18789/

The `keys/` directory holds the EC2 SSH key and any GCP service-account JSON — gitignored.

---

## Optional GUI

EC2 uses Ubuntu 22.04 LTS (server). To add desktop:

```bash
# SSH into EC2
sudo apt update && sudo apt install -y ubuntu-desktop
sudo reboot
```

---

## EC2 start/stop

```bash
# Stop (keeps EBS volume, no compute charges)
aws ec2 stop-instances --instance-ids <INSTANCE_ID>

# Start
aws ec2 start-instances --instance-ids <INSTANCE_ID>
```

The public DNS changes on each restart unless you attach an Elastic IP. After restart, refresh the host in `keys/help.md`.

Get the instance ID:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ea-agent" \
  --query 'Reservations[].Instances[].InstanceId' --output text
```

## Resize EBS volume (e.g. 20GB → 30GB)

```bash
aws ec2 modify-volume --volume-id <VOLUME_ID> --size 30
```

After modification, on the EC2:

```bash
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
```

(Adjust device name with `lsblk` if different.)

---

## Updates

```bash
cd ~/openclaw
git pull
docker compose build
docker compose up -d
```
