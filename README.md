# Configuration of Debian with UFW (firewall) and Nginx (web-server)


## Basic setup
> During the basic setup, you will need to configure the machine directly as access via terminal-window isn't possible yet.

1. Sign into the hypervisor, e.g. Proxmox.

1. Set up a new VM (nothing fancy, e.g. `1 core`, `1 processor`, `4096 KB` memory, `5 GB` storage) and install Debian.

1. Sign in using the root-user:
    - login: `root`
    - Password: {the password you specified for the root-user during installation}

1. Install "sudo" ("superuser do"), which allows other users to do superuser stuff:
    ```
    apt install sudo
    ```

1. Add your non-root user to the sudo group:
    ```
    usermod -aG sudo {username}
    ```

1. Enable SSH-server (if disabled), and enabled it on system start-up:
    ```
    sudo systemctl enable ssh
    sudo systemctl start ssh
    ```

1. Check the IP-address:
    ```
    sudo ip addr show
    ```
    
    This provides you with insight into the adapters and names, e.g. `ens18`, which you will need when setting a fixed IP-address next.

1. Set a static IP-address:
    ```
    sudo nano /etc/network/interfaces
    ```

    > If the "nano" editor isn't installed, you can install it by executing `sudo apt-get install nano`.
    
    Ensure that the "# The primary network interface" looks as follows:
    ```
    allow-hotplug ens18
    iface ens18 inet static
    address 192.168.0.249
    netmask 255.255.255.0
    gateway 192.168.0.1
    dns-nameservers 8.8.8.8 1.1.1.1
    ```

    > Note that the actual IP-address is the value of the `address` property - in this case `192.168.0.249` which we'll be using in the remainder of this document.

    Save the file by:
    ```
    ctrl x
    y
    enter/return
    ```

    Reboot the machine to make the changes take effect:
    ```
    reboot now
    ```

Hooray! You've now completed the basic setup of your Debian Linux-machine. Good job! You can now close the GUI to the hypervisor and continue the configuration using a terminal on your own machine, which enables you to copy/paste, scroll editor-windows without going insane etc.

To connect to the machine, open a terminal-window and connect to the machine via SSH using your non-root user:
    ```
    ssh {username}@192.168.0.249
    ```

Type `yes` to confirm that you want to continue connecting.

> For the remainder of this guide, we will assume that you're connected to the machine using the local terminal-window - unless explicitly stated otherwise.


### Enable access to the machine via hostname
If you want to eanble accessing the machine via its hostname, e.g. `ssh {username}@hostname.local`, it needs to support "multicast DNS". You can enable this by installing "Avahi" via the following command:
```
sudo apt-get install avahi-daemon
```

Reboot the machine for the changes to take effect:
```
reboot now
```

To check the status of "Avahi", execute the following command:
```
systemctl status avahi-daemon
```

> Note that even though the service is running, it will probably display `Status: "avahi-daemon 0.8 starting up`.


Once the machine is up and running again, you should be able to access the machine via its hostname, which was specified during the basic setup. To modify the hostname you can cange the values for `127.0.0.1` in the `/etc/hosts` file. To launch an editor ("nano") with the said file, execute the following command:
```
sudo nano /etc/hosts
```

After making the desired changes, exit the file by:
```
ctrl x
y
enter/return
```


## Installing and setting up "Uncomplicated Firewall" (UFW)
Before letting any traffic reach the machine, you should install and configure a firewall, and for this purpose we'll use "Uncomplicated Firewall" (UFW).

From a terminal window, sign into the machine via SSH (as detailed above). From here:

1. Install UFW:
    ```
    sudo apt install ufw
    ```

1. Set up default policies:
    ```
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    ```

1. Allow SSH-connections:
    ```
    sudo ufw allow ssh
    ```

    > You can also enable SSH by specifying the port instead, i.e. `sudo ufw allow 22`.

1. Enable UFW:
    ```
    sudo ufw enable
    ```

The firewall is now active. If you want to view the current rules, you can run the command:
```
sudo ufw status verbose
```

### Allowing other connections through the firewall
To allow other connections, e.g. web-traffic, you can run the following commands:

1. HTTP-traffic:
    ```
    sudo ufw allow http
    ```

1. HTTPS-traffic:
    ```
    sudo ufw allow https
    ```


## Installing and setting up Nginx (web-server)
From a terimal window, sign into the machine via SSH (as detailed above). From here:

1. Update the system:
    ```
    sudo apt update && sudo apt upgrade
    ```

    > Notice that commands can be chained together using two ampersand-characters (, i.e. `&&`).

1. Install Nginx:
    ```
    sudo apt install nginx
    ```

The Nginx-service should start automatically, but you can check its status with `sudo systemctl status nginx`.

By default, Nginx only supports plain HTTP-connections, so you can now access the machine on HTTP via a web-browser on `http://192.168.0.249` (using a web-browser which allows non-HTTPS-connections) - so Brave probably won't do, but Safari will (and possibly Chrome as well).

To view Nginx' default configuration, execute the following command:
```
sudo nano /etc/nginx/sites-available/default
```

### Enable SSL-support
To enable SSL-support for Nginx' websites, you first need to ensure that the Nginx-server is reachable via the domain(s) that you want to serve using HTTPS. Ideally this would entail configuring your DNS-provider (primary DNS-server(s) for your domain) having an A-record for the naked domain as well as a wildcard, e.g.:

1. A-record: `yourdomain.tld`
1. A-record: `*.yourdomain.tld`

... both pointing to your router's public IP-address, but if your DNS-provider doesn't support wildcards, you can add an A-record for each subdomain, e.g. `yourdomain.tld`, `www.yourdomain.tld`, `anothersubdomain.yourdomain.tld`.

> > To obtain your router's public IP-address, you can visit a website built for the purpose, e.g. https://whatismyipaddress.com.

> Note that you can use a dynamic DNS service, e.g. [FreeDNS](https://freedns.afraid.org), to keep domains related to your router if you don't have a public, static IP-address.

DNS-changes are not instantaneous, so you may have to implement the changes with your DNS-provider and then wait a few hours.

Once the A-records are active, which you can verify by opening a terminal-window and executing `ping yourdomain.tld`, `ping www.yourdomain.tld`, `anothersubdomain.yourdomain.tld` etc. and seeing that it resolves to your router's public IP-address you can:

1. Install Certbot:
    ```
    sudo apt install certbot python3-certbot-nginx
    ```

1. Generate an SSL-certificate for your site:
    ```
    sudo certbot --nginx -d yourdomain.tld -d www.yourdomain.tld -d anothersubdomain.yourdomain.tld
    ```

    > Note that all possible domains have to be specified when generating the SSL-certificate, in addition to the domains being available and the Nginx-server being reachable. Each domain is specified using the `-d {domain}` flag.

Certbot will now generate SSL-certificates for the specified domains, and will modify the Nginx configuration accordingly.


### Modifying the website
If you want to modify the default website for Nginx, you can execute the following command to open it in "nano":
```
sudo nano /var/www/html/index.nginx-debian.html
```

## Setting up reverse-proxies on Nginx
As in relation to SSL-support, before you can set up a reverse proxy, you first need to ensure that the Nginx-server is reachable via the domain(s) that you want to redirect.

> Note that merely having the naked domain, e.g. `yourdomain.tld`, point to your router's public IP-address doesn't mean that subdomains resolve to the same address. Hence, you either need to set up a wildcard-redirect to handle all subdomains, or you need to specify A-records for each subdomain.

> TODO: Revise the following to adhere to the official recommendations once we actually know what we're doing.

To add a reverse proxy configuration, open Nginx configuration, e.g.
```
sudo nano /etc/nginx/conf.d/yourdomain.conf
```

The file will contain one or more `server` blocks, and each block is a defintion to Nginx on how to handle a website.

> You can specify as many `server` blocks as you need.

Each server block basically consists of:
```
server {
    listen      80;                        # which port to listen to
    
    server_name subdomain.yourdomain.tld;  # site-binding(s)

    location / {
        proxy_pass http://path.to.local.server:80;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Optional logs
    acces_log /var/log/nginx/sitename_access.log
    acces_log /var/log/nginx/sitename_error.log    
}
```

Or, for SSL:
```
server {
    listen      443 ssl;                   # which port to listen to
    # ssl_certificate stuff
    #..
    #.
    
    server_name subdomain.yourdomain.tld;  # site-binding(s)

    location / {
        proxy_pass https://path.to.local.server;
        proxy_redirect off;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Optional logs
    acces_log /var/log/nginx/sitename_access.log
    acces_log /var/log/nginx/sitename_error.log    
}
```

> IMPORTANT: Make sure to specify the correct protocol for the `proxy_pass` property, i.e. `http` or `https`, as e.g. proxying to a HTTP-site which then redirects to HTTPS can result in a "too many redirects occurred" error.

After making changes to Nginx, restart the service for the changes to take effect:
```
sudo systemctl restart nginx
```

To see the configuration which Ngix is currently running, execute the following command:
```
sudo nginx -T
```

> Notice that the `T` flag is case-sensitive.


### Routing multiple ports
If you need to route multiple ports to a reverse-proxy host, you will need to specify multiple `server` blocks for the host - one for each port. Merely specifying that the `server` should `listen` to multiple ports, doesn't cause a request to be routed to the same port on the reverse-proxy host. I.e. if you want your (SSL, in this case) website to respond to requests on e.g. port 443 and 8338, then your setup could look like the following example:

```
server {
    listen      443 ssl;                   # which port to listen to
    # ssl_certificate stuff
    #..
    #.
    
    server_name subdomain.yourdomain.tld;  # site-binding(s)

    location / {
        proxy_pass https://path.to.local.server;
        proxy_redirect off;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Optional logs
    acces_log /var/log/nginx/sitename_access.log
    acces_log /var/log/nginx/sitename_error.log    
}

server {
    listen      8338 ssl;                   # which port to listen to
    # ssl_certificate stuff
    #..
    #.
    
    server_name subdomain.yourdomain.tld;  # site-binding(s)

    location / {
        proxy_pass https://path.to.local.server:8338;
        proxy_redirect off;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Optional logs
    acces_log /var/log/nginx/sitename_access.log
    acces_log /var/log/nginx/sitename_error.log    
}
```

> Notice that the first block which just uses the standard SSL-port, `443`, doesn't need a port-specification, e.g. `:443` for the `proxy_pass` value, but does need the port-specification for a non-standard port, e.g. `:8338`.

Remember to allow incoming connections to the port in the [firewall](#allowing-other-connections-through-the-firewall).
