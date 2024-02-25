# Redasu Install Memo

# Server Basic setup

### Vim

```bash
sudo apt-get update
sudo apt-get install -y vim
```

### SSH

Change port no.

```bash
sudo vi /etc/ssh/sshd_config
```

```ssh
Port XXXXX
Protocol 2
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitEmptyPasswords no
SyslogFacility AUTHPRIV
LogLevel VERBOSE
```

```bash
sudo /etc/init.d/ssh restart
```

### Firewall

if there is no module, install it.

```bash
sudo apt update
sudo apt install -y ufw
```

```bash
sudo ufw enable
sudo ufw logging on
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow XXXXX # set ssh port
sudo ufw allow from 172.128.128.0/24 to any port YYYYY # set db port if you need
sudo ufw reload
sudo ufw status
```

### Time zone

```bash
sudo apt update
sudo apt install -y tzdata
sudo tzselect # select Asis time zone
echo "TZ='Asia/Tokyo'; export TZ" >> ~/.bashrc
source ~/.bashrc
date
```

### Set NO automatic upgrede

```bash
sudo vi /etc/apt/apt.conf.d/20auto-upgrades
```

```
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
```

### Other

```bash
sudo mkdir /home/share
sudo chown -R ubuntu:ubuntu /home/share/
```

### Reboot

```bash
sudo reboot
```

# Local Network

```bash
sudo vi /etc/netplan/50-cloud-init.yaml
```

```
        eth1:
            addresses:
            - 172.128.128.10/24
            set-name: eth1
            match:
                macaddress: fa:16:3e:94:58:36
```

Previously, below config was set but default gateway config occur to default port forwarding when run "docker run -p XX:XX".
At that time, the connection was successful from only eth1, not eth0.

```diff
-            gateway4: 172.128.128.1
```

```bash
sudo netplan apply
ip a
```

# Redash

### Create Instance

```bash
sudo apt update
sudo apt install -y git
cd ~
git clone https://github.com/getredash/setup.git
cd ~/setup
vi setup.sh
```

Remove extra port forwarding.

```diff
  curl -OL https://raw.githubusercontent.com/getredash/setup/"$GIT_BRANCH"/data/docker-compose.yml
  sed -ri "s/image: redash\/redash:([A-Za-z0-9.-]*)/image: redash\/redash:$LATEST_VERSION/" docker-compose.yml
+ sed -i -e :loop -e 'N; $b loop' -e 's/ports:.*"5000:5000"//g' docker-compose.yml
+ sed -i -e 's/"80:80"/"22280:80"/g' docker-compose.yml
+ sed -i -e '/^\s*$/d' docker-compose.yml
  echo "export COMPOSE_PROJECT_NAME=redash" >>~/.profile
  echo "export COMPOSE_FILE=$REDASH_BASE_PATH/docker-compose.yml" >>~/.profile
```

```bash
bash setup.sh
```

### Network

If the server use container 

```bash
sudo vi /etc/ufw/sysctl.conf
```

```diff
-#net/ipv4/ip_forward=1
+net/ipv4/ip_forward=1
```

```bash
sudo vi /etc/default/ufw
```

```diff
-DEFAULT_FORWARD_POLICY="DROP"
+DEFAULT_FORWARD_POLICY="ACCEPT"
```

```bash
sudo vi /etc/ufw/before.rules
```

Add end of the file.

```diff
+*nat
+:POSTROUTING ACCEPT [0:0]
+:PREROUTING ACCEPT [0:0]
+-F
+-A POSTROUTING -s 172.128.64.0/24 -o docker0 -j MASQUERADE
+-A PREROUTING -p tcp --dport 55432 -s 172.128.64.0/24 -j DNAT --to-destination 172.17.0.2:5432
+COMMIT
```

```bash
sudo ufw allow from 172.128.64.0/24 to any port 55432 # set db port if you need
sudo ufw reload
sudo ufw status
```