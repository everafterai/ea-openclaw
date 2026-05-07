# 01 — AWS EC2 Instance

The EC2 instance for this project already exists. Connection details live in `keys/help.md`.

```
Host:  ec2-35-175-147-1.compute-1.amazonaws.com
User:  ubuntu
Key:   keys/open-claw.pem
```

```bash
ssh -i keys/open-claw.pem ubuntu@ec2-35-175-147-1.compute-1.amazonaws.com
```

Run from the project root so the relative key path resolves. `keys/` is gitignored.

**Reference:** [EC2 docs](https://docs.aws.amazon.com/ec2/) | [OpenClaw docs](https://docs.openclaw.ai/)

---

## Provisioning a fresh instance (only if recreating from scratch)

Skip this section if you're using the existing instance above.

Replace `<REGION>` with your region (e.g. `us-east-1`) and `<MY_IP>` with your public IP (`curl -s ifconfig.me`).

### 1. Create key pair

```bash
aws ec2 create-key-pair \
  --key-name open-claw \
  --query 'KeyMaterial' \
  --output text \
  --region <REGION> > keys/open-claw.pem

chmod 400 keys/open-claw.pem
```

### 2. Create security group (SSH from your IP)

```bash
aws ec2 create-security-group \
  --group-name ea-agent-sg \
  --description "ea-agent SSH" \
  --region <REGION>

aws ec2 authorize-security-group-ingress \
  --group-name ea-agent-sg \
  --protocol tcp --port 22 \
  --cidr <MY_IP>/32 \
  --region <REGION>
```

The OpenClaw gateway is reached via SSH tunnel, so no other inbound rules are needed.

### 3. Find latest Ubuntu 22.04 AMI

```bash
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text \
  --region <REGION>
```

### 4. Launch instance

```bash
aws ec2 run-instances \
  --image-id <AMI_ID> \
  --instance-type t3.small \
  --key-name open-claw \
  --security-groups ea-agent-sg \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=20,VolumeType=gp3}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ea-agent}]' \
  --region <REGION>
```

**Instance types:**
- `t3.small` — 2 vCPU, 2GB RAM (~$15/mo, plenty for the Minimus image flow — no source build)
- `t3.micro` — 2 vCPU, 1GB RAM (free tier eligible; tight, but works)
- `t3.medium` — 2 vCPU, 4GB RAM (~$30/mo; recommended once multiple internal users hit the gateway)

Get the public DNS:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ea-agent" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].PublicDnsName' \
  --output text \
  --region <REGION>
```

Update `keys/help.md` with the new host.

---

## Optional GUI

The Ubuntu AMI is server-only. To add a desktop later (after SSH in step 02):

```bash
sudo apt update && sudo apt install -y ubuntu-desktop
sudo reboot
```

---

**Next:** [02-OPENCLAW-DOCKER.md](./02-OPENCLAW-DOCKER.md)
