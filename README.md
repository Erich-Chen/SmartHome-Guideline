# SmartHome-Guideline
Set up a smart home system including homeassistant and homebridge,  based on Ubuntu 16.04LTS. 

## Prerequisite
* VMware Player
* Ubuntu Server 16.04.2 LTS  
** Network Adaptor: Bridged  
** hostname: smarthome  
** username: smarthome  
* 中国国内用户最好自备稳定的全局VPN，我没有对中国的网络环境做个别优化。  
* 认真推荐墨澜的系列文章，写得非常好： https://sspai.com/post/38849 http://cxlwill.cn/ 

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y openssh-server vim curl

# Create a new sudo user "smarthome"
sudo adduser smarthome
sudo usermod -aG sudo smarthome

# Change Timezone
sudo timedatectl set-timezone "Asia/Shanghai"

# Change hostname
sudo sed -i "s/.*/smarthome/g" /etc/hostname
sudo sed -i "s/127.0.1.1.*/127.0.1.1\tsmarthome/g" /etc/hosts
# Reboot to take effect
sudo reboot
```
NOTE: login as user 'smarthome' from now on.   

## Install homeassistant (easy way, NOT virtualenv)
 
```
sudo apt install -y python3 python3-pip 
sudo pip3 install --upgrade pip
sudo python3 -m pip install wheel
sudo pip3 install homeassistant

## if you prefer virtualenv: 
# sudo apt install -y python3-venv
# python3 -m venv hass_venv && cd hass_venv && source bin/activate
# python3 -m pip install --upgrade pip
# python3 -m pip install wheel
# python3 -m pip install homeassistant

# Test Run
hass --version

# run home assistant
hass

# solve missing of 'sqlalchemy' for HA v0.62.1
sudo pip3 install sqlalchemy

echo You will access via "http://$(echo $(hostname -I))::8123" on another computer with GUI under the same LAN. 

echo Press Ctrl+C, maybe two or three times, to terminal the task and get back for further confirguration. 

# Autostart home-assistant Using Systemd
cat << EOL | sudo tee /etc/systemd/system/hass.service
[Unit]
Description=Home Assistant
After=network-online.target

[Service]
Type=simple
User=%i
# ExecStart=$(which hass)
## for virtualenv:
ExecStart=$(which hass) -c "/home/smarthome/.homeassistant"

[Install]
WantedBy=multi-user.target

EOL

sudo systemctl --system daemon-reload
sudo systemctl enable home-assistant@smarthome
# sudo systemctl start|stop|restart home-assistant@smarthomes
```

## Install MQTT support
```
sudo apt install -y mosquitto mosquitto-clients
sudo pip3 install paho-mqtt    # was: python-mosquitto
sudo systemctl start mosquitto
```

## ~~Install homebridge~~ (HA supports HomeKit from 0.64) 
HA support HomeKit from version 0.64  
https://home-assistant.io/components/homekit/  
```
sudo apt install -y git make g++ curl
sudo apt install -y python
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt install -y nodejs
sudo apt install -y libavahi-compat-libdnssd-dev
sudo npm install -g --unsafe-perm homebridge hap-nodejs node-gyp

cd /usr/lib/node_modules/homebridge/
# If "No such directory" error, use below path instead: 
# cd /usr/local/lib/node_modules/homebridge/

sudo npm install --unsafe-perm bignum
# No worry on donwload ERRs, it is about windows system. 

cd /usr/lib/node_modules/hap-nodejs/node_modules/mdns
# If failed, use below instead
# cd /usr/local/lib/node_modules/hap-nodejs/node_modules/mdns

sudo node-gyp BUILDTYPE=Release rebuild

# Test Run HomeBridge
cd ~
homebridge

# Install plugs:
sudo npm install -g homebridge-homeassistant

# Create a sample config.json for homebridge
# username under bridge will be set as mac address of vm
cat << EOL | sudo tee ~/.homebridge/config.json
{
  "bridge": {
    "name": "Homebridge",
    "username": "$(echo $(cat /sys/class/net/e*/address) | awk '{print toupper($0)}')",
    "port": 51826,
    "pin": "031-45-154"
  },

  "platforms": [
    {
      "platform": "HomeAssistant",
      "name": "HomeAssistant",
      "host": "http://127.0.0.1:8123",
      "password": "",
      "supported_types": ["automation", "binary_sensor", "climate", "cover", "device_tracker", "fan", "group", "input_boolean", "light", "lock", "media_player", "remote", "scene", "script", "sensor", "switch", "vacuum"],
      "logging": false,
      "verify_ssl": false
    }
  ],

  "description": "This is an example configuration file with one fake accessory and one fake platform. You can use this as a template for creating your own configuration file containing devices you actually own."

}

EOL

# Fixes NPM permission folowwing error info
sudo chown -R $USER:$(id -gn $USER) /home/smarthome/.config

# Test Run, use CTRL+C to terminal
cd ~
homebridge


# Autostart homebridge Using Systemd
cat << EOL | sudo tee /etc/systemd/system/homebridge@smarthome.service
[Unit]
Description=Homebridge HomeKit Server 
After=network.target

[Service]
Type=simple
User=%i
ExecStart=$(which homebridge)

[Install]
WantedBy=multi-user.target

EOL

sudo systemctl --system daemon-reload
sudo systemctl enable homebridge@smarthome
```

## Reboot to Test Autostart
```
sudo reboot
# Get back to system, double check autostarts
sudo ps -ef | grep homeassistant@smarthome
sudo ps -ef | grep homebridge@smarthome
```

## Others
### Set an api password for homeassistant
```
# there are 4 files to edit to set up api_password for homeassistant. 
## 1. creates 'secrects.yaml'
vim /home/smarthome/.homeassistant/secrets.yaml   # secrects is plural
# api_password: SOME_PASSWORD
## 2. /home/smarthome/.homeassistant/configration.yaml
http:
  api_password: !secret api_password
## 3. /home/smarthome/.homebridge/config.json
      "password": "YOUR_HOMEASSISTANT_API_PASSWORD",
      "logging": true,
## 4. /home/smarthome/conf/appdaemon.yaml
  HASS:
    ha_key: "yourPassword"
```

~~### FRP- Fast Reverse Proxy (discarded)~~  
Refer to HTTPS.md  
https://github.com/fatedier/frp/blob/master/README_zh.md 
* Server 
```
# frps.ini
[common]
bind_port = 7000
vhost_http_port = 8123

# run
nohup frps -c frps.ini &
```
* Client
```
# frpc.ini
[common]
server_addr = xx.xx.xx.xx  # IP of FRPS server
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6022

[web]
type = http
local_port = 8123
custom_domains = xx.xx.xx.xx  # IP or URL

# run 
nohup frpc -c frpc.ini &
```

### SSL  

### Replace Database 
ref. http://cxlwill.cn/Home-Assistant/HomeAssistant-PostgreSQL/  
```
echo "deb http://apt.postgresql.org/pub/repos/apt/ xenial-pgdg main" | sudo tee  -a /etc/apt/sources.list
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc |   sudo apt-key add -

sudo apt update
sudo apt install -y postgresql-10 postgresql-server-dev-10
psql --version
pg_config --version
sudo -u postgres createuser smarthome
sudo -u postgres createdb -O smarthome homeassistant

pip3 install psycopg2
## virtualenv:
# cd ~/homeassistant/ && source bin/activate
# pip3 install psycopg2

vim ~/.homeassistant/configuration.yaml
## add below content
# recorder:
#   db_url: postgres://@/homeassistant
```

### MagicMirror  
ref. https://github.com/MichMich/MagicMirror  
```
cd ~

curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt install -y nodejs

git clone https://github.com/MichMich/MagicMirror
cd ~/MagicMirror
npm install
node serveronly

# edit whitelist
cp ~/MagicMirror/config/config.js.sample ~/MagicMirror/config/config.js
sudo vim ~/MagicMirror/config/config.js
# address: "0.0.0.0",
# ipWhitelist: [],

# install 

# autostart using systemd
cat << EOL | sudo tee /etc/systemd/system/magicmirror@smarthome.service
[Unit]
Description=Magic Mirror
After=network-online.service

[Service]
Environment=NODE_PORT=8080
Type=simple
User=%i
WorkingDirectory=/home/smarthome/MagicMirror/
ExecStart=/usr/bin/node serveronly/

[Install]
WantedBy=multi-user.target

EOL

sudo systemctl --system daemon-reload
sudo systemctl enable magicmirror@smarthome
```

## configuration.yaml 
I forked cxlwill's HA-config:   
https://github.com/cxlwill/HA_config    
https://github.com/cxlwill/HA_config  


## Upgrade
```
# System Update
sudo apt update && sudo apt upgrade -yy

# homeassistant upgrade
cd ~/homeassistant && source bin/activate
python3 -m pip install --upgrade homeassistant

# Node/npm Update
sudo npm cache clean -f
sudo npm install -g n
sudo n latest
sudo npm install -g npm

# Homebridge Update
sudo npm update -g homebridge

# Magic Mirror Update
cd ~/MagicMirror
git pull && npm install
```  

## Configuration Update
```
# To update homebridge config, first stop homebridge service
sudo systemctl stop homebridge@smarthome.service
vim ~/.homebridge/config.json
# IMPORTANT: remove cached persist and accessories information, which will be rebuilt after service restart
rm -rf ~/.homebridge/accessories/ ~/.homebridge/persist/
# Start homebridge service again
sudo systemctl start homebridge@smarthome.service
# in HomeKit on iPhone, remove homebridge and add back. 
```

# Notepad++ and NPP FTP plugin   
I use Notepad++ and a plugin "NPP FTP" to access and edit configuration files. I do not feel like semba a secure enough.    
