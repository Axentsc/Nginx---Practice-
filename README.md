Setup
Step 1 — Install Nginx on web1 and web2

sudo apt update && sudo apt install nginx -y
systemctl enable nginx

 web1
echo '<h1>Hello from web1</h1>' | sudo tee /var/www/html/index.html

 web2
echo '<h1>Hello from web2</h1>' | sudo tee /var/www/html/index.html

 verify
curl localhost

Step 2 — Configure Nginx on gateway
Create /etc/nginx/sites-available/loadbalancer:

upstream backend {
  ip_hash;  # sticky sessions
  server 192.168.56.10 max_fails=3 fail_timeout=30s;
  server 192.168.56.11 max_fails=3 fail_timeout=30s;
}

server {
  listen 80;
  location / {
    proxy_pass http://backend;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}

nginx -t  # check config
ln -s /etc/nginx/sites-available/loadbalancer /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
systemctl reload nginx


Step 3 — Configure HAProxy
Edit /etc/haproxy/haproxy.cfg:
haproxyfrontend http_front
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /health
    server web1 192.168.56.10:80 check inter 5s fall 3 rise 2
    server web2 192.168.56.11:80 check inter 5s fall 3 rise 2

  Step 4 — Add /etc/hosts and verify
bash# add to /etc/hosts
192.168.56.10   web1
192.168.56.11   web2

 test
curl http://web1
curl http://web2

 test load balancing — should alternate between web1 and web2
for i in {1..6}; do curl http://web1; echo; done

# Monitoring script 

#!/bin/bash
set -Eeuo pipefail

SERVICE="nginx"
LOG_FILE="/var/log/service_monitor.log"
DATE=$(date "+%Y-%m-%d %H:%M:%S")
WEB1="http://web1"
WEB2="http://web2"

if systemctl is-active --quiet $SERVICE; then
    echo "$DATE - $SERVICE is running" >> $LOG_FILE
else
    echo "$DATE - $SERVICE is NOT running. Restarting..." >> $LOG_FILE
    systemctl restart $SERVICE
    if systemctl is-active --quiet $SERVICE; then
        echo "$DATE - $SERVICE restarted successfully" >> $LOG_FILE
    else
        echo "$DATE - FAILED to restart $SERVICE" >> $LOG_FILE
        exit 2
    fi
fi

for URL in $WEB1 $WEB2; do
    if curl --fail --silent --max-time 5 $URL > /dev/null; then
        echo "$DATE - $URL is ok" >> $LOG_FILE
    else
        echo "$DATE - $URL is DOWN" >> $LOG_FILE
    fi
done
