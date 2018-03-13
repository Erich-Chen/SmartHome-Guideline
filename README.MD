# SmartHome Guideline on Raspberry Pi

## Prerequisite
* Raspberry Pi 3 Model B  
Install Raspbian with Etcher or NOOBS.  
create a file `ssh` in root folder of NOOBS or /boot of raspbian can enable SSH if run Raspberry Pi totally headless.  

## Basic Setup
** hostname: smarthome  
** username: smarthome  
* 中国国内用户最好自备稳定的全局VPN，我没有对中国的网络环境做个别优化。  
* 认真推荐墨澜的系列文章，写得非常好： https://sspai.com/post/38849 http://cxlwill.cn/ 

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y openssh-server vim curl

# Configure password, timezone, hostname, keyboard, locale (en_US.UTF8), etc. (Optional)
sudo raspi-config

# If prefer commandline
## Change Timezone
# sudo timedatectl set-timezone "Asia/Shanghai"

## Change hostname
# sudo sed -i "s/.*/smarthome/g" /etc/hostname
# sudo sed -i "s/127.0.1.1.*/127.0.1.1\tsmarthome/g" /etc/hosts

## Change locale
# sudo localectl set-local LANG="en_US.UTF-8"

## Edit WiFi 
# sudo vim /etc/wpa_supplicant/wpa_supplicant.conf

```


## Install homeassistant (virtualenv)
 
```
sudo apt install -y python3 python3-pip python3-venv
python3 -m pip install --upgrade pip
python3 -m venv homeassistant && cd homeassistant && source bin/activate
python3 -m pip install wheel
python3 -m pip install homeassistant

# Test Run
hass --version

# test run home assistant
hass
## It may take a while for first time start. Wait until 
## Do not miss error or warning during home-assistant start, fix it if any. 
## Try uncomment the line `# discovery` if it takes too long when there is a lot of devices in your LAN. 
echo You will access via "http://$(echo $(hostname -I))::8123" with browser in same LAN. 
echo Press Ctrl+C, maybe two or three times, to terminal the task and get back for further confirguration. 

# Autostart home-assistant Using Systemd
cat << EOL | sudo tee /etc/systemd/system/hass.service
[Unit]
Description=Home Assistant
After=network-online.target

[Service]
Type=simple
User=$USER
ExecStart=$(which hass) -c "$HOME/.homeassistant"

[Install]
WantedBy=multi-user.target

EOL

sudo systemctl --system daemon-reload
sudo systemctl enable hass.service
sudo systemctl start hass.service
sudo systemctl status hass.service

# sudo systemctl start|stop|restart hass.service
```

Homeassistant supports HomeKit from verion 0.64, so HomeBridge is not needed any more. 

## Replace Database 
A better database can improve startup speed of HA
```
deactivate && cd ~  # Exit python venv
sudo apt update
sudo apt install -y postgresql-9.6 postgresql-server-dev-9.6
psql --version
pg_config --version
sudo -u postgres createuser pi
sudo -u postgres createdb -O pi homeassistant
cd ~/homeassistant && source bin/activate
pip3 install psycopg2

vim ~/.homeassistant/configuration.yaml
## add below content (uncomment)
# recorder:
#   db_url: postgres://@/homeassistant

# A little update to systemd to make sure postgresql starts before HA
sudo vim /etc/systemd/system/hass.service
# After=network-online.target postgresql.service

```


## Set an api password for homeassistant
```
## 1. creates / edit 'secrects.yaml'
vim ~/.homeassistant/secrets.yaml   # secrects is plural
# http_password: SOME_PASSWORD      # uncomment this line
## 2. /home/smarthome/.homeassistant/configration.yaml   # uncomment below lines
# http:
#   api_password: !secret http_password

```

## Port Forwarding with SSS tunnel


## SSL  



### MagicMirror  
ref. https://github.com/MichMich/MagicMirror  
Easy Install script:  
`bash -c "$(curl -sL https://raw.githubusercontent.com/MichMich/MagicMirror/master/installers/raspberry.sh)"`

Manual Install:   
```
deactivate && cd ~
# Install node.js 8.x LTS
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt install -y nodejs
git clone https://github.com/MichMich/MagicMirror
cd ~/MagicMirror
npm install
# Test run
node serveronly

# Optional (NOT SAFE): Edit whitelist to allow access from remote 
cp ~/MagicMirror/config/config.js.sample ~/MagicMirror/config/config.js
vim ~/MagicMirror/config/config.js
# address: "0.0.0.0",
# port: 8234,
# ipWhitelist: [],

# autostart using systemd
cat << EOL | sudo tee /etc/systemd/system/magicmirror.service
[Unit]
Description=Magic Mirror
After=network-online.service

[Service]
Environment=NODE_PORT=8234
Type=simple
User=$USER
WorkingDirectory=$HOME/MagicMirror/
ExecStart=/usr/bin/node serveronly/

[Install]
WantedBy=multi-user.target

EOL

sudo systemctl --system daemon-reload
sudo systemctl enable magicmirror
```



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


# Magic Mirror Update
cd ~/MagicMirror
git pull && npm install
```  

# Notepad++ and NPP FTP plugin   
I use Notepad++ and a plugin "NPP FTP" to access and edit configuration files. I do not feel like semba a secure enough.    
