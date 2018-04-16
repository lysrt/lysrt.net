---
title: "Using Caddy on Linux with systemd"
date: 2018-01-03T10:58:26+01:00
---

You've probably heard about [Caddy](https://caddyserver.com/). If you haven't, you should definitely check it out.  
Caddy is an amazing web server, with automatic HTTPS. It uses HTTP/2 by default, and is so simple to configure.

If you host your own website on a dedicated server, at some point you had to decide which web server to use. Generally it will be `Apache HTTP Server` or `NGINX`.  
They do the work, sure. What if you need your custom API to run besides your website? Use a *reverse proxy*.
If you have several domain names pointing to the same IP address? Use *Virtual Hosts*.
What if you need HTTPS? HTTP/2 post? And if you need to pull your website from a Git repository?

Don't get me wrong, Apache and NGINX can do all that, I just think they are hard to configure.

This article will be about installing and configuring Caddy on Linux, and making it a systemd daemon.

# Caddy configuration

5 minutes were all I needed to grow a strong interest for Caddy.  
Today it has simplified my workflow so much, I would struggle to use another web server. 

Here is what my Caddyfile looks like for `lysrt.net`:

```
lysrt.net {
    gzip
    root /path/to/lysrt.net/
    tls my@email.com
    
    proxy /api localhost:8080 {
        without /api
        transparent
    }
    
    log stdout
    errors {
        log stderr
        404 404.html
    }
}
```

That's all.

As you see, it has the following features:

* gzip compression
* Automated HTTPS using [Let's Encrypt](https://letsencrypt.org/)
* A reverse proxy from `lysrt.net/api` to `localhost:8080`
* Customizable error pages and standard outputs (more on this below)

*The rest of this article focuses on Linux only.*

# Installing Caddy on Linux

The easiest way is to use the provided installation script: https://getcaddy.com/ (using `curl` or `wget`). For instance to get a non commercial license, with the dns and http.git plugins:  

```shell
$ curl https://getcaddy.com | bash -s personal http.git,dns
```

If you don't feel like running `curl xxx | bash`, here is the standard way of doing.

First, get Caddy on the [official download page](https://caddyserver.com/download). You can select your license type, and any plugins you need. In my case I got the following file: `caddy_v0.10.11_linux_amd64_custom_personal.tar.gz`.

Let's extract it to a `caddy` directory and move it to the install path, `/usr/local/bin` in my case:
```shell
$ tar -xzf caddy*.tar.gz caddy
$ mv ./caddy /usr/local/bin
```

`/usr/local/bin/` should be in your PATH, so you can check caddy has been successfully installed:
```shell
$ caddy -version
Caddy 0.9.5
```

# Running caddy as a systemd service

For me, the prefered way of running Caddy is a a daemon, or systemd service.  
The main advantage is I don't have to worry about it, it will always be running on my server. And the logs are managed by `journalctl`, which is super convenient.

Here is a quick summary of the workflow. For detailed instructions check the page [systemd Service Unit for Caddy](https://github.com/mholt/caddy/tree/master/dist/init/linux-systemd).

Set appropriate permissions to caddy
```shell
$ sudo chown root:root /usr/local/bin/caddy
$ sudo chmod 755 /usr/local/bin/caddy

$ sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/caddy
```

If they don't exist, create the group and user `www-data`
```shell
$ sudo groupadd -g 33 www-data
$ sudo useradd \
  -g www-data --no-user-group \
  --home-dir /var/www --no-create-home \
  --shell /usr/sbin/nologin \
  --system --uid 33 www-data
```

Create config directories and Caddyfile
```shell
$ sudo mkdir /etc/caddy
$ sudo chown -R root:www-data /etc/caddy
$ sudo mkdir /etc/ssl/caddy
$ sudo chown -R root:www-data /etc/ssl/caddy
$ sudo chmod 0770 /etc/ssl/caddy

$ sudo touch /etc/caddy/Caddyfile
$ sudo chown www-data:www-data /etc/caddy/Caddyfile
$ sudo chmod 444 /etc/caddy/Caddyfile
```

Create home directory for your server (the sites location)
```shell
$ sudo mkdir /var/www
$ sudo chown www-data:www-data /var/www
$ sudo chmod 555 /var/www

$ sudo mkdir /var/www/example.com
$ sudo chown -R www-data:www-data /var/www/example.com
$ sudo chmod -R 555 /var/www/example.com
```

Configure the Caddyfile accordingly
```
example.com {
    root /var/www/example.com
    ...
}
```

Configure `caddy.service` as a systemctl daemon
```shell
$ wget https://raw.githubusercontent.com/mholt/caddy/master/dist/init/linux-systemd/caddy.service
$ sudo cp caddy.service /etc/systemd/system/
$ sudo chown root:root /etc/systemd/system/caddy.service
$ sudo chmod 644 /etc/systemd/system/caddy.service
$ sudo systemctl daemon-reload
$ sudo systemctl start caddy.service
```

Automatically start Caddy on boot
```shell
$ sudo systemctl enable caddy.service
```

# Updating Caddy

Just replace the executable with the latest version and restart the service:
```shell
$ sudo systemctl restart caddy.service
```

# Updating Caddy's configuration with live reload

After making updates to the Caddyfile, you can reload the configuration with no downtime:
```shell
sudo pkill -USR1 caddy
```

# Managing logs

Using `log stdout` and `errors stderr` in your Caddyfile, will make logs and errors available through systemd journaling, `journalctl`.  
It your distrib doesn't use systemd, logfiles should be in `/var/log`.

To see logs for the current boot of the service:
```shell
$ journalctl --boot -u caddy.service
```

To follow the latest logs in real time:
```shell
$ journalctl -f -u caddy.service
```



Feb 25 16:08:40 lys systemd[1]: Reloading Caddy HTTP/2 web server.

Feb 25 16:08:40 lys caddy[2529]: 2018/02/25 16:08:40 [INFO] SIGUSR1: Reloading
Feb 25 16:08:40 lys caddy[2529]: 2018/02/25 16:08:40 [INFO] Reloading

Feb 25 16:08:40 lys systemd[1]: Reloaded Caddy HTTP/2 web server.

Feb 25 16:08:41 lys caddy[2529]: From https://github.com/lysrt/lysrt.net
Feb 25 16:08:41 lys caddy[2529]: * branch            master     -> FETCH_HEAD
Feb 25 16:08:41 lys caddy[2529]: Already up-to-date.
Feb 25 16:08:41 lys caddy[2529]: 2018/02/25 16:08:41 https://github.com/lysrt/lysrt.net.git pulled.
Feb 25 16:08:41 lys caddy[2529]: 2018/02/25 16:08:41 [INFO] Reloading complete




Feb 25 16:09:15 lys caddy[2529]: 2018/02/25 16:09:15 [INFO] SIGUSR1: Reloading
Feb 25 16:09:15 lys caddy[2529]: 2018/02/25 16:09:15 [INFO] Reloading

Feb 25 16:09:16 lys caddy[2529]: From https://github.com/lysrt/lysrt.net
Feb 25 16:09:16 lys caddy[2529]: * branch            master     -> FETCH_HEAD
Feb 25 16:09:16 lys caddy[2529]: Already up-to-date.
Feb 25 16:09:16 lys caddy[2529]: 2018/02/25 16:09:16 https://github.com/lysrt/lysrt.net.git pulled.
Feb 25 16:09:16 lys caddy[2529]: 2018/02/25 16:09:16 [INFO] Reloading complete
