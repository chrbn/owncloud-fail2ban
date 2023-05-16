# Protect your OCIS deployment with Fail2Ban behind Caddy as a reverse proxy
This guide describes a way to protect your OCIS deployment with Fail2Ban when using Caddy as a reverse proxy to serve your users.

Note that this guide expects you already have a running OCIS deployment exposed to the Internet through Caddy acting as a reverse proxy.

The commands provided below assume you are using a Debian-based or Ubuntu-based Linux system with systemd.

# Install Fail2Ban
On your server, run the following commands to install Fail2Ban if you don't already have it installed.

```Shell
sudo apt update
sudo apt install fail2ban
```

# Enable logs in your Caddyfile
Edit your Caddy file (usually located in /etc/caddy).

```sudo nano /etc/caddy/Caddyfile```

Locate the configuration for your OCIS deployment, and enable logging by adding a `log` block.

```
your_ocis_domain.ext {
        reverse_proxy :LOCAL_PORT

        log {
                output file /var/log/caddy/ocis.log
                format console
        }
}
```

Basically, it enables Caddy logs for your OCIS deployment and will log all HTTP requests Caddy forwards to OCIS.

# Adding Fail2Ban configuration

## Filter
We are going to add a filter to Fail2Ban so it knows what to look for in your Caddy log file.

Create the following file and content:
```
sudo nano /etc/fail2ban/filter.d/caddy-ocis-login.conf
```
```
[Definition]
failregex = ^.*"remote_ip": "<HOST>".*"method": "POST".*"uri": "\/signin\/v1\/identifier/_/logon".*$
ignoreregex =
```
Exit and save the modifications.

When you try to authenticate on OCIS, the web UI or client app will POST an HTTP request to the /signin/v1/\_/logon endpoint.
It means if we have many requests to this endpoint in our Caddy log file, maybe someone is trying to bruteforce its way to your OCIS deployment.

## Jail
We are going to create a second file indicating to Fail2Ban what do to when the filter we created above is triggered.

Create the following file and content:
```
sudo nano /etc/fail2ban/jail.d/caddy-ocis-login.conf
```

```
[caddy-ocis-login]
port    = http,https
logpath = /var/log/caddy/ocis.log
enabled = true
banTime = 300
findTime = 300
maxretry = 5
```

Make sure to keep the same filenames and contents. If you name your files with something else than `caddy-ocis-login`, you need to update the jail name (between [ and ]) with the name you used.

Here, Fail2Ban will look for the filter defined above in a 5 minutes window in the Caddy logs. If it finds it 5 times in a 5 minutes window, Fail2Ban will ban the associated client IP during 5 minutes, assuming someone failed to authenticate 5 times in less than 5 minutes. We want to delay further attempts to avoid bruteforce attacks.

Of course, you can tweak these settings to reflect what you need.

## Restart Fail2Ban
Restart Fail2Ban so it takes into account your new filter and jail.

```
sudo systemctl restart fail2ban.service
```

Now, your OCIS deployment is protected against buteforce attacks on its authentication endpoint.
