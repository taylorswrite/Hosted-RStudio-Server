# Hosted-RStudio-Server

Use DuckDNS to host your computers RStudio-Server

### Checklist

- [ ] Router capable of port-forwarding (Can't use a Spectrum router)

- [ ] Ubuntu-based system

- [ ] Open firewall ports

- [ ] Setup DuckDNS domain name

- [ ] Configure Nginx

- [ ] Configure Certbot


### Host System Setup

#### Install Rstudio Server

Visit the [RStudio Server download page](https://posit.co/download/rstudio-server/). Select the Ubuntu tab then select `Ubuntu 12 / Ubuntu 22`. Follow the instructions.

1) Ensure that you successfully installed rstudio-Server.

```
sudo systemctl status rstudio-server
```

2) If rstudio-server is not running, run the following command to start it.

```
sudo systemctl start rstudio-server
```

3) Run the command below to start rstudio-server at boot-up.

```
sudo systemctl enable rstudio-server
```

#### Install Necessary Packages

- `ddclient`: Controls the domain name.

- `nginx`: Connects rstudio-server to the correct ports

- `certbot`: Obtains a HTTPS certificate.

- `python3-certbot-nginx`: A `certbot` package on for `nginx`

```
sudo apt-get update
sudo apt-get install ddclient nginx certbot python3-certbot-nginx
```

### Domain Name Configuration

#### Router Configuration

- Find your router manufacturer (Spectrum Routers can't forward ports). 

- Search "how to forward-ports on a [manufacturer] router"

- Open 2 ports (External:Internal):

  - 80:80 Used for http request (We will later forward those to HTTPS)

  - 443:443 Used for https request


#### Get a Free Domain Name

- Create an account on [DuckDNS](https://www.duckdns.org/).

- Make a domain name.

  - http://somethingcool.duckdns.org

- **IMPORTANT** Above the created domain save the following

  - account

  - token

### Configure Host Network

Use `sudo vi` or `sudo nano` to edit configuration files

#### Create SSL Certificates

```
sudo mkdir /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 \
-newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key \
-out /etc/nginx/ssl/nginx.crt
```

#### Configure Domain Name Control

```
sudo vi /etc/ddclient.conf 
```

Add the following text to ddclient.conf.

Replace somethingcool.duckdns.org with your domain name

Replace your_duckdns_token with your token form DuckDNS

```
daemon=300
syslog=yes
pid=/var/run/ddclient.pid

protocol=dyndns2
use=web, web=checkip.duckdns.org/, web-skip='IP Address'
server=www.duckdns.org
login=somethingcool.duckdns.org
password=your_duckdns_token
```

```
sudo systemctl start ddclient
sudo systemctl enable ddclient
```
#### Configure Nginx

```
sudo vi /etc/nginx/conf.d/rstudio.conf
```

- Add the text below to `/etc/nginx/conf.d/rstudio.conf`. 
- Replace `somethingcool.duckdns.org` (2 occurances) with the domain name you created in DuckDNS.

```
server {
    listen 80; 
    server_name somethingcool.duckdns.org;
    return 301 https://$server_name$request_uri;  # Redirect to HTTPS
}

server {
    listen 443 ssl;
    server_name somethingcool.duckdns.org;

    # https public (.crt) and private key (.key)
        ssl_certificate /etc/nginx/ssl/nginx.crt;
        ssl_certificate_key /etc/nginx/ssl/nginx.key;

    location / {
        proxy_pass http://127.0.0.1:8787; 
    }
}
```

Then edit the `/etc/nginx/nginx.conf` file.

```
sudo vi /etc/nginx/nginx.conf
```

Add the text below to the bottom of the http block. Example below

```
        # All you other settings up here... 
        server_names_hash_bucket_size 128;

        map $http_upgrade $connection_upgrade {
                default upgrade;
                ''      close;
        }
```

Example:
```
http {
        # All you other settings up here... 
        server_names_hash_bucket_size 128;

        map $http_upgrade $connection_upgrade {
                default upgrade;
                ''      close;
        }
}
```

#### Configure rstudio-server

Edit /etc/rstudio/rserver.conf configuration file.

```
sudo vi /etc/rstudio/rserver.conf
```

Add the line below to `/etc/rstudio/rserver.conf`.

```
www-address=127.0.0.1
```


Optional: Edit /etc/rstudio/rsession.conf configuration file to remove timeout.

```
sudo vi /etc/rstudio/rsession.conf
```

Add the following line to `/etc/rstudio/rsession.conf`.

```
session-timeout-minutes=0
```


#### Obatian https Certificates

Restart everything to confirm changes are implemented.

```
sudo rstudio-server restart
sudo systemctl restart nginx
```

Obtain https from a trusted source, `certbot`

Replace somethingcool.duckdns.org

Slowly follow directions and answer questions.

select duckdns, input account (from DuckDNS) and token (from DuckDNS), and enter na for organization. 
```
sudo certbot --nginx -d somethingcool.duckdns.org
```


Sources:
[Stack Overflow: How can I set up an rstudio server to run with ssl on aws](https://stackoverflow.com/questions/53102584/how-can-i-set-up-an-rstudio-server-to-run-with-ssl-on-aws)
