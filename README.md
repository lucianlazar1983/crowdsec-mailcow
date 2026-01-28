# **Securing Mailcow with CrowdSec via Docker API**

This guide provides instructions for implementing CrowdSec to protect a Mailcow: dockerized installation. It utilizes direct Docker log acquisition, which eliminates the need for complex rsyslog configurations or manual file monitoring.

## **1. Install CrowdSec**

#### Install the CrowdSec security engine using the official repository script.
```bash
curl -s https://install.crowdsec.net | sudo bash  
sudo apt-get install crowdsec crowdsec-firewall-bouncer-iptables  (Debian based distro)
or 
sudo yum/dnf install crowdsec crowdsec-firewall-bouncer-iptables (Redhat based distro)
```


## **2. Install Required Collections and Parsers**

To process Mailcow logs, you must install the service-specific collections and the Docker log parser.

#### Install service collections  
```bash
sudo cscli collections install crowdsecurity/nginx  
sudo cscli collections install crowdsecurity/postfix  
sudo cscli collections install crowdsecurity/dovecot

```

#### Install the Docker parser  
```bash
sudo cscli parsers install crowdsecurity/docker-logs
```


#### Restart CrowdSec to load components  
```bash
sudo systemctl restart crowdsec
```


## **3. Configure Data Acquisition**

Configure CrowdSec to monitor specific containers by editing `/etc/crowdsec/acquis.d/mailcow.yaml`

Note: Dovecot and Postfix logs inside Mailcow containers follow a syslog-like format. 
Labeling them as syslog allows CrowdSec to strip metadata automatically. 
Nginx logs use a standard format and should be labeled as nginx.

#### Contents of  `/etc/crowdsec/acquis.d/mailcow.yaml`
```bash
source: docker  
container_name:  
  - mailcowdockerized-dovecot-mailcow-1  
  - mailcowdockerized-postfix-mailcow-1  
labels:  
  type: syslog  
---  
source: docker  
container_name:  
  - mailcowdockerized-nginx-mailcow-1  
labels:  
  type: nginx
```


#### Apply changes:
```bash
sudo systemctl restart crowdsec
```


## **4. Verify the Pipeline**

Use cscli explain to confirm logs are captured and parsed.

### **Dovecot Verification**
```bash
docker logs mailcowdockerized-dovecot-mailcow-1 --tail 10 | cscli explain --type syslog -f -
```

Check for: `crowdsecurity/dovecot-logs status: Success`

### **Postfix Verification**
```bash
docker logs mailcowdockerized-postfix-mailcow-1 --tail 100 | grep -E "reject:|warning:|authentication failed" | tail -n 1 | cscli explain --type syslog -f -
```

Check for: `crowdsecurity/postfix-logs status: Success`

### **Nginx Verification**
```bash
docker logs mailcowdockerized-nginx-mailcow-1 --tail 10 | cscli explain --type nginx -f -
```

Check for: `crowdsecurity/nginx-logs status: Success`


## **5. Monitoring and Metrics**

#### View real-time processing statistics:
```bash
cscli metrics
```

#### List active decisions and bans:
```bash
cscli decisions list
```

## **6. Whitelisting Internal Traffic**

Create a whitelist to prevent CrowdSec from blocking internal Mailcow communication (e.g., Rspamd or Nginx proxying). 

Create `/etc/crowdsec/parsers/s02-enrich/mailcow-whitelist.yaml:`
```bash
name: my docker-whitelist  
description: "Whitelist internal Mailcow traffic"  
whitelist:  
  reason: "Internal Mailcow communication"  
  ip:  
    - "127.0.0.1"
    - "::1"
  cidr:  
    - "172.22.1.0/24" # Adjust to match your IPv4 docker network
    - "fd4d:6169:6c63:6f77::/64" # Adjust to match your IPv6 docker network
    - "172.16.0.0/12" # Standard Docker network

```

Restart the service after:
```bash
sudo systemctl restart crowdsec  
```

Done. 
