# Installation nouvelle OpenStack avec devstack

1. System Preparation and User Creation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git
sudo useradd -s /bin/bash -d /opt/stack -m stack
sudo chmod +x /opt/stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo -u stack -i   
```

2. Clone and Configure DevStack

```bash
git clone https://opendev.org/openstack/devstack -b stable/2026.2
cd devstack   
```

Create a local.conf file in the devstack directory to define passwords and network settings. A minimal configuration looks like this:

```conf
[[local|localrc]]
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
HOST_IP=<YOUR_SERVER_IP>   
```
3. Run Installation

```bash
./stack.sh   
```
