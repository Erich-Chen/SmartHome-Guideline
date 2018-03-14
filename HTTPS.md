## Preconditions  
1. I have a vultr server (cost me only USD2.5 per month) with IP address 45.32.xx.xx;  
2. I have a free domain `mydomain.tk` registered at http://www.freenom.com/ and A-record point to vultr server (45.32.xx.xx);  
3. My home-assistant server does not have public IP.  

### Portforwarding
on Server(Vultr Server): 
```
echo 'GatewayPorts clientspecified' | sudo tee -a /etc/ssh/sshd_config
sudo systemctl restart sshd.service
```

on Client (HA server) - Quick Test
```
ssh -NR 0.0.0.0:8123:127.0.0.1:8123 root@45.32.xx.xx &
## -R a:p1:b:p2的含义：
## 打开服务器上的p1端口，向a开放（a为0.0.0.0或*代表所有网络，a为127.0.0.1代表本机），
## 这个端口上的所有网络流量会通过ssh隧道转发到客户端，并由客户端转发到b地址的p2端口。
```

on Client (HA server)
```
# Generate SSH key to login without asking for password
ssh-keygen -o -a 100 -t ed25519 -P ''
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@45.32.xx.xx

# Install `autossh` to keep SSH tunnel always alive
sudo apt install -y autossh

cat << EOL | sudo tee /etc/systemd/system/autossh@smarthome.service
[Unit]
Description=Keeps a SSH tunnel to my vultr server open and alive
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/autossh \\
                -i "/home/smarthome/.ssh/id_ed25519" \\
                -M 0 -N -q \\
                -o "ServerAliveInterval 60" \\
                -o "ServerAliveCountMax 3" \\
                -R 0.0.0.0:80:127.0.0.1:80 \\
                -R 0.0.0.0:443:127.0.0.1:8123 \\
                -R 0.0.0.0:8080:127.0.0.1:8080 \\
                root@45.32.74.141
# -M 0 --> no monitoring
# -N Just open the connection and do nothing (not interactive)
# -R IPs_Allowed:Port_on_RemoteServer:LocalHost:Local_Port
# forward port 80 for letsenrypt certbot; port 443 for https access of home assistant; 8080 for http access of MagicMirror
# must use root user on remote server to allow well-known ports (80/443/etc.) forwarding

[Install]
WantedBy=multi-user.target

EOL

sudo systemctl --system daemon-reload
sudo systemctl enable autossh@smarthome
```

## HTTPS access
```
mkdir ~/certbot && cd $_
wget https://dl.eff.org/certbot-auto && chmod a+x certbot-auto
./certbot-auto certonly --standalone --preferred-challenges http-01 --email my@email.com -d mydomain.tk -d www.mydomain.tk

sudo chmod 755 /etc/letsencrypt/live/
sudo chmod 755 /etc/letsencrypt/archive/

sudo vim ~/.homeassistant/config.yaml
# edit below lines: 
# http:
#   api_password: YOUR_PASSWORD
#   ssl_certificate: /etc/letsencrypt/live/mydomain.tk/fullchain.pem
#   ssl_key: /etc/letsencrypt/live/mydomain.tk/privkey.pem
#   base_url: mydomain.tk

```
