# Project Iris EC2 Deployment

Follow these steps in this exact order.

## 1. Open AWS EC2

1. Sign in to AWS Console
2. Search for `EC2`
3. Open `EC2`
4. Click `Launch instance`

## 2. Fill the Launch Instance screen

### Name

In `Name and tags`:

- Name: `project-iris-ec2`

### AMI

In `Application and OS Images (Amazon Machine Image)`:

- Choose `Ubuntu`

### Instance type

In `Instance type`:

- Choose an instance type marked `Free tier eligible`

### Key pair

In `Key pair (login)`:

1. Click `Create new key pair`
2. Key pair name: `project-iris-key`
3. Key pair type: keep default
4. Private key file format: `.pem`
5. Click `Create key pair`
6. The file will download to your machine

### Network settings

In `Network settings`:

1. Click `Edit`
2. Under `Security group`, choose `Create security group`
3. Security group name: `project-iris-sg`
4. Description: `Security group for Project Iris`

Add the first inbound rule:

- Type: `SSH`
- Port range: `22`
- Source type: `Anywhere-IPv4`

AWS will fill:

```text
0.0.0.0/0
```

Use this because GitHub Actions must connect to your EC2 server over SSH.

Add the second inbound rule:

1. Click `Add security group rule`
2. Type: `Custom TCP`
3. Port range: `8501`
4. Source type: `Anywhere`

AWS will fill:

```text
0.0.0.0/0
```

This allows the app to open in the browser.

### Launch

After filling all of the above:

1. Click `Launch instance`

## 3. Copy the public IP

After the instance starts:

1. Open `EC2`
2. Click `Instances`
3. Click `project-iris-ec2`
4. Copy `Public IPv4 address`

You will use this later as:

```text
EC2_HOST=YOUR_EC2_PUBLIC_IP
```

## 4. Move the key file and test SSH

Run these commands on your own machine:

```bash
mv ~/Downloads/proj-iris-keys.pem ~/.ssh/proj-iris-keys.pem
chmod 400 ~/.ssh/proj-iris-keys.pem
ssh -i ~/.ssh/proj-iris-keys.pem ubuntu@34.205.139.54
```

Replace:

- `YOUR_EC2_PUBLIC_IP` with the public IP you copied from AWS

If login works, leave the server by running:

```bash
exit
```

## 5. Add GitHub secrets

Open your GitHub repository.

Go to:

`Settings` -> `Secrets and variables` -> `Actions`

Create these secrets:

### `EC2_HOST`

Value:

```text
YOUR_EC2_PUBLIC_IP
```

### `EC2_USERNAME`

Value:

```text
ubuntu
```

### `EC2_SSH_KEY`

Run:

```bash
cat ~/.ssh/project-iris-key.pem
```

Copy the full output and paste it into the secret.

It must include:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

Do not paste:

- only part of the key
- the file path
- the file name

## 6. Push the code

Run these commands in this project folder:

```bash
git add .
git commit -m "Set up EC2 deployment"
git push origin main
```

If Git says there is nothing to commit, run:

```bash
git push origin main
```

## 7. Wait for deployment

In GitHub:

1. Open `Actions`
2. Open `Deploy To AWS EC2`
3. Wait until it shows success

## 8. Open the app

Open this in your browser:

```text
http://YOUR_EC2_PUBLIC_IP:8501
```

## 9. If something fails

SSH into the server:

```bash
ssh -i ~/.ssh/project-iris-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Check service status:

```bash
sudo systemctl status project-iris --no-pager
```

Check logs:

```bash
journalctl -u project-iris -n 100 --no-pager
```
