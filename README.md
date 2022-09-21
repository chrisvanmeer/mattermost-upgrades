# MATTERMOST UPGRADE TEST

## PRE

```bash
multipass launch --name matter --cpus 2 --mem 1G --disk 16G --cloud-init cloud-init.yaml focal
```

### packages

```bash
sudo apt update && sudo apt install -y mariadb-server && sudo apt dist-upgrade -y
```

### database

```mysql
sudo mysql
create user 'mmuser'@'%' identified by 'mmuser-password';
create database mattermost;
grant all privileges on mattermost.* to 'mmuser'@'%';
exit
```

### binary

```bash
mkdir -p mm/dist/v{6.1.3,6.2.5,7.3.0}
wget https://releases.mattermost.com/6.1.3/mattermost-team-6.1.3-linux-amd64.tar.gz -P /home/chris/mm/dist/v6.1.3/
wget https://releases.mattermost.com/6.2.5/mattermost-team-6.2.5-linux-amd64.tar.gz -P /home/chris/mm/dist/v6.2.5/
wget https://releases.mattermost.com/7.3.0/mattermost-team-7.3.0-linux-amd64.tar.gz -P /home/chris/mm/dist/v7.3.0/
```

## INSTALL v.6.1.3

### files

```bash
cd /home/chris/mm/dist/v6.1.3
tar -xzf mattermost*.gz
sudo mv mattermost /opt
sudo mkdir /opt/mattermost/data
sudo useradd --system --user-group mattermost
sudo chown -R mattermost:mattermost /opt/mattermost
sudo chmod -R g+w /opt/mattermost
sudo vi /opt/mattermost/config/config.json
```

### config

```bash
sudo sed -i 's/"DriverName": "postgres",/"DriverName": "mysql",/g' /opt/mattermost/config/config.json
sudo sed -i 's#"DataSource": "postgres://mmuser:mostest@localhost/mattermost_test?sslmode=disable\\u0026connect_timeout=10",#"DataSource": "mmuser:mmuser-password@tcp(localhost:3306)/mattermost?charset=utf8mb4,utf8\\u0026writeTimeout=30s",#' /opt/mattermost/config/config.json
```

### test

```bash
cd /opt/mattermost
sudo -u mattermost ./bin/mattermost
```

### stop

`ctrl+c`

### systemd

```bash
echo """[Unit]
Description=Mattermost
After=network.target
After=mysql.service
BindsTo=mysql.service

[Service]
Type=notify
ExecStart=/opt/mattermost/bin/mattermost
TimeoutStartSec=3600
KillMode=mixed
Restart=always
RestartSec=10
WorkingDirectory=/opt/mattermost
User=mattermost
Group=mattermost
LimitNOFILE=49152

[Install]
  WantedBy=mysql.service""" | sudo tee /lib/systemd/system/mattermost.service
```

```bash
sudo systemctl daemon-reload
sudo systemctl status mattermost.service
sudo systemctl start mattermost.service
curl http://localhost:8065
sudo systemctl enable mattermost.service
```

## CONFIG

Configure some settings in MM to populate it.

## UPGRADE

### upgrade to v.6.2.5

```bash
cd /home/chris/mm/dist/v6.2.5
tar -xf mattermost*.gz --transform='s,^[^/]\+,\0-upgrade,'
sudo systemctl stop mattermost
cd /opt
sudo cp -ra mattermost/ mattermost-back-$(date +'%F-%H-%M')/
sudo find mattermost/ mattermost/client/ -mindepth 1 -maxdepth 1 \! \( -type d \( -path mattermost/client -o -path mattermost/client/plugins -o -path mattermost/config -o -path mattermost/logs -o -path mattermost/plugins -o -path mattermost/data \) -prune \) | sort | sudo xargs rm -r
sudo cp -an /home/chris/mm/dist/v6.2.5/mattermost-upgrade/. mattermost/
sudo chown -R mattermost:mattermost mattermost
sudo systemctl start mattermost
```

### upgrade to v7.3.0

```bash
cd /home/chris/mm/dist/v7.3.0
tar -xf mattermost*.gz --transform='s,^[^/]\+,\0-upgrade,'
sudo systemctl stop mattermost
cd /opt
sudo cp -ra mattermost/ mattermost-back-$(date +'%F-%H-%M')/
sudo find mattermost/ mattermost/client/ -mindepth 1 -maxdepth 1 \! \( -type d \( -path mattermost/client -o -path mattermost/client/plugins -o -path mattermost/config -o -path mattermost/logs -o -path mattermost/plugins -o -path mattermost/data \) -prune \) | sort | sudo xargs rm -r
sudo cp -an /home/chris/mm/dist/v7.3.0/mattermost-upgrade/. mattermost/
sudo chown -R mattermost:mattermost mattermost
sudo systemctl start mattermost
```
