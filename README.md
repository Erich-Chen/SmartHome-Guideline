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
exit
```
###NOTE: You will Exit and login again as user 'smarthome'  

```
# Change Timezone
sudo timedatectl set-timezone "Asia/Shanghai"

# Change hostname
sudo sed -i "s/.*/smarthome/g" /etc/hostname
sudo sed -i "s/127.0.1.1.*/127.0.1.1\tsmarthome/g" /etc/hosts
# Reboot to take effect
sudo reboot
```

## Install homeassistant (easy way, NOT virtualenv)
* 虚拟机代价很低，专VM专用，就不必使用virtualenv来隔离生产环境了
```
sudo apt install -y python3 python3-pip python3-venv
sudo pip3 install --upgrade pip
sudo pip3 install homeassistant

## if you prefer virtualenv:
# python3 -m venv homeassistant
# cd homeassistant
# source bin/activate
# pip install --upgrade pip
# python3 -m pip install homeassistant

# Test Run
hass --version
echo You will access via "http://$(echo $(hostname -I))::8123" on another computer with GUI under the same LAN. 
echo Press Ctrl+C, maybe two or three times, to terminal the task and get back for further confirguration. 
hass

# Autostart home-assistant Using Systemd
cat << EOL | sudo tee /etc/systemd/system/home-assistant@smarthome.service
[Unit]
Description=Home Assistant
After=network-online.target

[Service]
Type=simple
User=%i
ExecStart=$(which hass)
## for virtualenv:
# ExecStart=$(which hass) -c "/home/smarthome/.homeassistant"

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

## Install homebridge
```
sudo apt install -y git make g++ curl
sudo apt install -y python
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt install -y nodejs
sudo apt install -y libavahi-compat-libdnssd-dev
sudo npm install -g --unsafe-perm homebridge hap-nodejs node-gyp

cd /usr/lib/node_modules/homebridge/
# If "No such directory" error, use below: 
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
      "supported_types": ["fan", "garage_door", "input_boolean", "light", "lock", "media_player", "rollershutter", "scene", "switch"],
      "logging": false,
      "verify_ssl": false
    }
  ],

  "description": "This is an example configuration file with one fake accessory and one fake platform. You can use this as a template for creating your own configuration file containing devices you actually own."

}

EOL

# Fixing NPM permission
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

## AppDaemon (Dashboard)
```
sudo pip3 install appdaemon
mkdir ~/conf
touch ~/conf/appdaemon.yaml

cat << EOL | sudo tee /home/smarthome/conf/appdaemon.yaml
AppDaemon:
  logfile: STDOUT
  errorfile: STDERR
  threads: 10
  cert_verify: False
  
HASS:
  ha_url: http://127.0.0.1:8123
  ha_key: 

HADashboard:
  dash_url: http://127.0.0.1:5050
  dash_force_compile: 1

# Apps
hello_world:
  module: hello
  class: HelloWorld

EOL

mkdir /home/smarthome/conf/apps/

cat << EOL | sudo tee /home/smarthome/conf/apps/hello.py
import appdaemon.appapi as appapi

#
# Hello World App
#
# Args:
#

class HelloWorld(appapi.AppDaemon):

  def initialize(self):
     self.log("Hello from AppDaemon")
     self.log("You are now ready to run Apps!")

EOL

mkdir /home/smarthome/conf/dashboards/

cat << EOL | sudo tee /home/smarthome/conf/dashboards/Hello.dash
#
# Main arguments, all optional
#
title: Hello Panel
widget_dimensions: [120, 120]
widget_margins: [5, 5]
columns: 8

label:
    widget_type: label
    text: Hello World

layout:
    - label(2x2)
EOL


# Test Run
appdaemon -c ~/conf


# Autostart AppDaemon Using Systemd

cat << EOL | sudo tee /etc/systemd/system/appdaemon@smarthome.service
[Unit]
Description=AppDaemon
After=home-assistant@smarthome.service

[Service]
Type=simple
User=%i
ExecStart=$(which appdaemon) -c /home/smarthome/conf

[Install]
WantedBy=multi-user.target

EOL

sudo systemctl --system daemon-reload
sudo systemctl enable appdaemon@smarthome
```

## Reboot to Test Autostart
```
sudo reboot
# Get back to system, double check autostarts
sudo ps -ef | grep homeassistant@smarthome
sudo ps -ef | grep homebridge@smarthome
```

## Others
### Set up api password for homeassistant
```
# there are 4 files to edit to set up api_password for homeassistant. 
## 1. touch /home/smarthome/.homeassistant/secrets.yaml   # secrects is plural
api_password: SOME_PASSWORD
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

### FRP- Fast Reverse Proxy  
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
http://cxlwill.cn/Home-Assistant/HomeAssistant-PostgreSQL/ 

## configuration.yaml 
I forked cxlwill's HA-config: https://github.com/cxlwill/HA_config  
https://github.com/cxlwill/HA_config  


## Upgrade
```
# System Update
sudo apt update && sudo apt upgrade -yy

# homeassistant upgrade
sudo pip3 install --upgrade homeassistant

# Node/npm Update
sudo npm cache clean -f
sudo npm install -g n
sudo n latest
sudo npm install -g npm

# Homebridge Update
sudo npm update -g homebridge

# AppDaemon Update
sudo pip3 install --upgrade appdaemon

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
