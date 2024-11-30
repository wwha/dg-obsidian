---
{"dg-publish":true,"permalink":"/add-fail2-ban-protection-for-services-using-cloudflare-on-mac/","created":"2024-11-10T16:59:24.197+08:00","updated":"2024-11-30T09:03:35.397+08:00"}
---

Using [Fail2Ban](https://github.com/fail2ban/fail2ban) could protect public services, like VaultWarden, from attackers brutal force logins. It monitors log files and bans IP addresses conducting too many failed login attempts. 

If the services use Cloudflare tunnel, the login requests IP addresses are from Cloudflare. Banning this IP address could not stop attackers. Fortunately, Cloudflare provides configuration to send the original login attempts IP address to the service and RESTAPI of firewall that could ban IP addresses. Then, Fail2Ban could do its job.

## Install and Configure Fail2Ban
- Install Fail2Ban on Mac: `brew install fail2ban`. 
	Configuration directory: `/opt/homebrew/etc/fail2ban`.
	Binary directory: `/opt/homebrew/opt/fail2ban/bin/fail2ban-client`.
	Log directory: `/opt/homebrew/var/log/fail2ban.log`. It is defined in `fail2ban.conf` from configuration directory.
- Create a new jail: `vim /opt/homebrew/etc/fail2ban/jail.local`.
	- Add the following to the `jail.local`.
	```
	[cf-vwlogin]
	
	enabled = true
	port = http,https
	filter = cf-vwlogin
	chain = FORWARD
	logpath = /Users/*/docker-data/vaultwarden-server/data/vwarden.log
	banaction = cloudflare-apiv4
				pf[actiontype=<allports>]
	maxretry = 3
	bantime = 14400
	findtime = 14400
	```
	
	- It defines a `cf-vwlogin` jail, which monitors http and https ports, scans `vwarden.log`. If unauthorized login attempts are over 3 times during 14400 s, Fail2Ban would execute the `banaction` defined in `cloudflare-apiv4` and `pf` for 14400 s.
	- On Mac OS, `iptables` is not available. The default is pf (packet filter), a powerful firewall used for filtering TCP/IP traffic. Just add pf in `banaction` to enable it.
- Create a filter: `vim /opt/hombrew/etc/fail2ban/filter.d/cf-vwlogin.local`.
	- Add the following to `vwlogin.local`, which  from [vaultwarden-Fail2Ban Setup](https://github.com/dani-garcia/vaultwarden/wiki/Fail2Ban-Setup#filter).
	```
	[INCLUDES]
	before = common.conf
	
	[Definition]
	failregex = ^.*?Username or password is incorrect\. Try again\. IP: <ADDR>\. Username:.*$
	ignoreregex =
	```
- Create an action: `cp /etc/homebrew/action.d/cloudflare.conf /etc/homebrew/action.d/cloudflare-apiv4.local`.
	- In `cloudflare-apiv4.local`, the `actionban` and `actionunban` is curl command using Cloudflare RESTAPI to request Cloudflare firewall to ban and unban the IP address. Add your Cloudflare email address and Cloudflare API Key for `cfuser` and `cftoken`, respectively.
- Restart fail2ban: `brew service restart fail2ban`.
## Configure NGINX

To include the original login requests IPs, configure the NGINX based on [Cloudflare Docs](https://developers.cloudflare.com/support/troubleshooting/restoring-visitor-ips/restoring-original-visitor-ips/#nginx-1).
- Create a file for Cloudflare CDN IPs: `vim /opt/homebrew/etc/nginx/cf-realip.conf`, and add Cloudflare's IPs from https://www.cloudflare.com/ips.
```
set_real_ip_from 173.245.48.0/20
set_real_ip_from 103.21.244.0/22
set_real_ip_from 103.22.200.0/22
set_real_ip_from 103.31.4.0/22
set_real_ip_from 141.101.64.0/18
set_real_ip_from 108.162.192.0/18
set_real_ip_from 190.93.240.0/20
set_real_ip_from 188.114.96.0/20
set_real_ip_from 197.234.240.0/22
set_real_ip_from 198.41.128.0/17
set_real_ip_from 162.158.0.0/15
set_real_ip_from 104.16.0.0/13
set_real_ip_from 104.24.0.0/14
set_real_ip_from 172.64.0.0/13
set_real_ip_from 131.0.72.0/22
set_real_ip_from 2400:cb00::/32
set_real_ip_from 2606:4700::/32
set_real_ip_from 2803:f800::/32
set_real_ip_from 2405:b500::/32
set_real_ip_from 2405:8100::/32
set_real_ip_from 2a06:98c0::/29
set_real_ip_from 2c0f:f248::/32
```
- Add the following to `/opt/homebrew/etc/nginx/nginx.conf`,
```
##
# CF Real IP
##
include ./conf.d/cf-realip.conf;
real_ip_header X-Forwarded-For;
```
- Restart NGINX with: `brew services restart nginx`
## Test Fail2Ban Protection
- Check the jail status with: `sudo fail2ban-client status cf-vwlogin`
	```
	Status for the jail: cf-vwlogin
	|- Filter
	|  |- Currently failed: 0
	|  |- Total failed:     0
	|  `- File list:        /Users/docker-data/vaultwarden-server/data/vwarden.log
	`- Actions
	   |- Currently banned: 0
	   |- Total banned:     0
	   `- Banned IP list:
	```

- Trigger the Fail2Ban jail with wrong passwords. The login page would display the Access denied info. The visitor from that IP would not visit the login page for 14400 s.
	![asserts/5020ca36eb0bf5c29e61720252a2abb9_MD5.jpeg](/img/user/asserts/5020ca36eb0bf5c29e61720252a2abb9_MD5.jpeg)
	![asserts/84f2b895b343424edf12cf69a89d1345_MD5.jpeg](/img/user/asserts/84f2b895b343424edf12cf69a89d1345_MD5.jpeg)
	```
	sudo fail2ban-client status cf-vwlogin

	Status for the jail: cf-vwlogin
	|- Filter
	|  |- Currently failed: 0
	|  |- Total failed:     9
	|  `- File list:        /Users/feiyuan/docker-data/vaultwarden-server/data/vwarden.log
	`- Actions
	   |- Currently banned: 1
	   |- Total banned:     1
	   `- Banned IP list:   2406:*:*7930:4e6b
	sudo pfctl -a "f2b/cf-vwlogin" -t f2b-cf-vwlogin -Ts

	No ALTQ support in kernel
	ALTQ related functions disabled
    2406:da18
	```
- Unblock banned IPs with: `fail2ban-client set cf-vwlogin unbanip <IP>`. This would remove the IP from the jail and Cloudflare Firewall. The visitor from that IP would visit the login page again.
## Q&A
- The unblock banned IPs command does not remove the banned ip.
	The curl command of actionunban uses `jq`. The curl command fails since `jq` is not installed, without any error info. After installing `jq` with `brew install jp`, the curl command works good.
	```
	actionunban = id=$(curl -s -X GET <_cf_api_prms> \
                    "<_cf_api_url>?mode=block&configuration_target=<cftarget>&configuration_value=<ip>&page=1&per_page=1&notes=Fail2Ban%%20<name>" \
                    | { jq -r '.result[0].id' 2>/dev/null || tr -d '\n' | sed -nE 's/^.*"result"\s*:\s*\[\s*\{\s*"id"\s*:\s*"([^"]+)".*$/\1/p'; })
               if [ -z "$id" ]; then echo "<name>: id for <ip> cannot be found"; exit 0; fi;
               curl -s -o /dev/null -X DELETE <_cf_api_prms> "<_cf_api_url>/$id"

	 _cf_api_url = https://api.cloudflare.com/client/v4/user/firewall/access_rules/rules
	 _cf_api_prms = -H 'X-Auth-E
	```



## Reference

- https://ariq.nauf.al/blog/fail2ban-with-cloudflare/