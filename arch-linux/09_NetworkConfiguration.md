# Network Configuration

> This segment configures the following:
> networkmanager to use iwd as the backend
> openresolv for DNS

## IWD Backend for NM

```ini
# /etc/NetworkManager/conf.d/iwd.conf
[device]
wifi.backend=iwd

# /etc/NetworkManager/conf.d/openresolv.conf
[main]
dns=default
rc-manager=resolvconf
```

## DNS

```ini
# /etc/resolvconf.conf
name_servers="94.140.14.14 94.140.15.15"
name_servers_append="1.1.1.1 1.0.0.1"
```

> These use AdGuard DNS as primary and Cloudflare as fallback
