**goaccess command for real-time reports from nginx logs**

Preparation:

```bash
mkdir goaccess
cd goaccess
mkdir data
wget https://download.db-ip.com/free/dbip-city-lite-2024-01.mmdb.gz
gunzip dbip-city-lite-2024-01.mmdb.gz
sudo mkdir -p /var/www/yourdomain.com/static/goaccess/yoursecret/
sudo ln -s ./report.html /var/www/yourdomain.com/static/goaccess/yoursecret/index.html
```

The command to run goaccess by docker:

```bash
sudo tail -F /var/log/nginx/access.log | docker run 
  -p 7890:7890  # ws port
  -v `pwd`/dbip-city-lite-2024-01.mmdb:/dbip.mmdb  # to enable geoip, download by the link: https://download.db-ip.com/free/dbip-city-lite-2024-01.mmdb.gz
  -v `pwd`/data:/data  # the place to store the data parsed data
  -i -e LANG=$LANG allinurl/goaccess  # image name
  -a -o html --log-format COMBINED  # formating flags 
  --ws-url wss://yourdomain.com:443/goaccess/yoursecret/ws # specify the address of ws
  --enable-panel GEO_LOCATION  # to enable geoip
  --geoip-database /dbip.mmdb  # to enable geoip
  --restore --persist --db-path /data  # to store the results in files and use them to restore
  --real-time-html  # to handle real-time logs
  - > report.html
```

The nginx config to handle requests to goaccess:

```nginx
server {
  # some default config
  root /var/www/yourdomain.com/static;
  index index.html;
  
  location / {
    default_type "text/html";
    try_files $uri.html $uri $uri/ /index.html;
  }

  location ~ ^/goaccess/yoursecret/ws/?$ {
    proxy_connect_timeout 1d;
    proxy_send_timeout 1d;
    proxy_read_timeout 1d;
    proxy_pass http://127.0.0.1:7890;
  }

  # some default config
}
```
