# Nginx---Practice-

## 📌 Description
Сють практики поднять 3 виртуальные машини настроить 
на gateaway reverse proxy / load balancing / sticky session / healthy checks

## 🚀 Features
- reverse proxy
- load balancing
- sticky session
- chealthy checks

## ⚙️ Tech Stack
- Nginx
- Linux
- VirtualBox
- HaProxy

## 🧠 How it works
Gateaway выполняет функцию load balancing sticky session chealthy checks
и распределяет работу между двумя другими Nginx

## ▶️ How to run
Set up 3 machines on the same network 
sudo apt update && sudo apt upgrade - 3 VirtualBox
sudo apt install nginx -y
- Only for machine web 1 , web 2
systemctl enable nginx
echo <h1>"Hello from web1"</h1> | sudo tee /var/www/html/index.html - the first machine
echo <h1>"Hello from web2"</h1> | sudo tee /var/www/html/index.html - the second machine
Check if works - curl localhost

In gateaway 

nano /etc/nginx/sites-avaible/loadbalancer
upstream backend {
  ip_hash;
  server 192.168.56.10;
  server 192.168.56.11;

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
nginx -t - check for good configuration
ln -s /etc/nginx/sites-avaible /etc/nginx/sites-enabled/
rm -r /etc/nginx/sites-enabled/default
systemctl nginx reload 

sudo apt install haproxy -y
nano /etc/haproxy/haproxy.cfg

frontend http_front
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /health     # active health check
    server web1 192.168.1.11:80 check inter 5s fall 3 rise 2
    server web2 192.168.1.12:80 check inter 5s fall 3 rise 2
