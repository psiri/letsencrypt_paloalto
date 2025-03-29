# letsencrypt_paloalto
LetsEncrypt certificates for your Palo Alto Networks Firewalls! Can be adapted to work with most vendor makes/models.

For a detailed guide, please refer to the following: https://www.bitbodyguard.com/articles/palo-alto-networks/letsencrypt-certificates-for-palo-alto-networks-globalprotect-vpn/


# LetsEncrypt Certificates for Your Firewalls!

Have you wanted to take advantage of free LetsEncrypt certificates for your firewalls, VPN Portals, or network infrastructure? Did the short LetsEncrypt renewal cycle deter you? Perhaps you hit a roadblock automating the process without native LetsEncrypt support on-device? Maybe you are just tired of paying for your certificates?

If you answered “yes” to either of these questions, this is the script for you!

The following will demonstrate you how you can automate the LetsEncrypt certificate process for network and security devices. While I will focus on Palo Alto Networks firewalls for the purpose of this demonstration, the following can be adapted to work with almost any network device, and is particularly useful for VPN headend devices (regardless of make and model).

# Prerequisites

### Before you get started, you will require the following:
+ A linux-based container, VM, or host with reachability to your device’s management interface and the following packages installed:
  + openssl
  + pan-python
  + certbot
  + (optional, if you are using CloudFlare) certbot-dns-cloudflare
  ```
  sudo apt-get install python-pip certbot openssl 
  sudo pip install pan-python
  sudo pip install certbot-dns-cloudflare
  ```
  
+ An account with a certbot-compatible DNS provider.
  + A list of certbot DNS plugins can be found here:  https://certbot.eff.org/docs/using.html#dns-plugins
  + I will use CloudFlare in this example.  The dns-cloudflare plugin automates the process of a dns-01 challenge by creating and removing TXT records using the CloudFlare API
    + If you use CloudFlare, you will be required to pass in your CloudFlare credentials as a certbot argument.  By default, this is a .ini file containing your CloudFlare username and API key.
    + To obtain your CloudFlare API key, navigate to your CloudFlare admin panel and select “My Profile” from the upper-right corner.
    + Navigate to the “API Tokens” tab.  Select “View” next to “Global API Key”.  Copy this key into a .cloudflare.ini file.     + It is best practice to ensure this file can only be accessed by your user (or the user cron runs as).
    ```
    dns_cloudflare_email = psiri@bitbodyguard.com
    dns_cloudflare_api_key = 0123456789abcdef0123456789abcdef01234567
    ```
 + An API key for your firewall:
   + To generate an API key, you can use the following command:
   ```
   panxapi.py -h PAN_MGMT_IP_OR_FQDN -l USERNAME:'PASSWORD' -k
   ```
 + (Recommended) store your API key in a .panrc file:
   + As a good practice, avoid storing your API key directly in your script:
   ```
   panxapi.py -h PAN_MGMT_IP_OR_FQDN -l USERNAME:'PASSWORD' -k >> ~/.panrc
   ```
 
 ## Other Considerations
 
 If this linux instance is not behind the same public IP that the FQDN will resolve to, you may need to create a NAT rule on your firewall.  Certbot assumes that the certificate will be installed on the host issuing the call. While most linux based web servers make this process easy, network devices typically do not.
 
 + If your linux instance is not behind the same public IP as your VPN Portal/Gateway, you can create a NAT rule to ensure LetsEncrypt “sees” this host coming from the same public IP.
 + If your instance is NAT’d to the same public IP that your GlobalProtect Portal/Gateway uses, you can skip this step
 + If you are using a dns-provider plugin that handles domain validation for you, you can skip this step.
 
 # Initial Configuration - CloudFlare
To setup certbot initially, issue the following command.  This command will generate certificates non-interactively, automatically running a standalone web server for authentication and accepting the ToS.  While we can certainly generate and/or renew interactively, the ultimate goal is unattended automation.

+ Replace *.bitbodyguard.com with the desired certificate FQDN or a comma-separated list of domains.
  + *Note: The first domain provided will be the subject CN on the certificate! All other domains will appear as SANs*
+ Replace “/home/psiri/.cloudflare.ini” with the full path to your stored CloudFlare Credentials
+ In this example, I am requesting a wildcard certificate, so I will use “*.bitbodyguard.com”
```
cloudflare-credentials /home/psiri/.cloudflare.ini -d *.bitbodyguard.com --preferred-challenges dns-01
```

 # Initial Configuration - RFC 2136
If your DNS server has RFC 2136 configured, you can use the rfc2136 variant of the script to generate certificates.
  ```
  sudo apt install certbot python3-certbot-dns-rfc2136
  ```
rfc2136.ini configuration example:
```
# Target DNS server (IPv4 or IPv6 address, not a hostname)
dns_rfc2136_server = <YOUR_DNS_SERVER_IP>
# Target DNS port
dns_rfc2136_port = 53
# TSIG key name
dns_rfc2136_name = <KEY_NAME>
# TSIG key secret
dns_rfc2136_secret = <KEY_SECRET>
# TSIG key algorithm
dns_rfc2136_algorithm = HMAC-SHA512
# TSIG sign SOA query (optional, default: false)
dns_rfc2136_sign_query = false
```

# Initial Configuration - Standalone
### If you are not using CloudFlare DNS for authentication, you can optionally use the standalone method to perform initial certbot setup:

+ Replace *.bitbodyguard.com with the desired certificate FQDN or a comma-separated list of domains.
  + *Note: The first domain provided will be the subject CN on the certificate! All other domains will appear as SANs*
+ In this example, I am requesting a wildcard certificate, so I will use “*.bitbodyguard.com”
+ Replace <USERNAME>@<YOUR-DOMAIN> with your email address.  This is used for important account notifications.

```
certbot certonly -d *.bitbodyguard.com -m <USERNAME>@<YOUR-DOMAIN> --standalone -n --agree-tos
```

# The Script
The script is located in the file "pan_certbot" --> https://github.com/psiri/letsencrypt_paloalto/edit/master/pan_certbot

# How It Works
### This small bash script performs the following actions:
1. Uses the provided CloudFlare credentials file to call certbot’s dns-cloudflare plugin, automatically validating ownership of the FQDN/domain by creating and deleting TXT records
2. Accepts LetsEncrypt’s ToS and renews the certificate(s) for the provided FQDN(s)
3. Randomly generates a certificate passphrase using “openssl rand”
4. Creates a temporary, password-protected PKCS12 cert file named “letsencrypt_pkcs12.pfx” from the individual private and public keys issued by LetsEncrypt.
5. Uploads the temporary PKCS12 file to the firewall using the randomly-generated passphrase.  Certificate name is set to variable $CERT_NAME
6. Deletes the temporary PKCS12 certificate from linux host
7. (Optionally) Sets the certificate used within the GlobalProtect Portal’s SSL/TLS profile to the name of the new LetsEncrypt certificate
8. (Optionally) Sets the certificate used within the GlobalProtect Gateway’s SSL/TLS profile to the name of the new LetsEncrypt certificate
9. Commits the candidate configuration (synchronously) and reports for the commit result


# Automated Renewal and Installation
We can into a cronjob which will automatically renew and upload the certificates, modify the SSL/TLS Service Profiles (if required), and commit the configuration.

To make this a bit more adaptable to different scenarios, I have included the following variables:
+ *CLOUDFLARE_CREDS*: The full path to the “.cloudflare.ini” file
+ *CERT_NAME*: The name you wish to give the certificate on the device (Palo Alto Networks GUI:  Device –> Certificate Management –> Certificates)
+ *GP_PORTAL_TLS_PROFILE*: The name of the GlobalProtect SSL/TLS Service Profile used on the Portal.
+ *GP_GW_TLS_PROFILE*: The name of the GlobalProtect SSL/TLS Service Profile used on the Gateway. For single Portal/Gateway deployments using a single SSL/TLS profile, this may be the same as “GP_PORTAL_TLS_PROFILE”.

### Notes
+ As best-practice, you should use separate SSL/TLS Service Profiles for each Portal and Gateway.  This script assumes you have followed best-practices, but will also work with single-profile configurations.
+ With Palo Alto Networks Firewalls specifically, updating the SSL/TLS Service Profiles is only required when the name of the certificate referenced by the SSL/TLS Service Profile changes.
+ If the SSL/TLS Service Profiles have been updated with the name of the LetsEncrypt certificate (and you do not plan on timestamping or otherwise changing the certificate name), no modifications are required when a certificate is renewed. If you choose to append a timestamp or rename the certificate, you will need to programmatically update the SSL/TLS Service profiles.
  + To update the SSL/TLS Service Profile(s), uncomment lines 20 (for a single SSL/TLS Profile) and 22 (for two SSL/TLS Profiles)
  ```
  panxapi.py -h $PAN_MGMT -K $API_KEY -S "$CERT_NAME" "/config/shared/ssl-tls-service-profile/entry[@name='$GP_PORTAL_TLS_PROFILE']"
  panxapi.py -h $PAN_MGMT -K $API_KEY -S "$CERT_NAME" "/config/shared/ssl-tls-service-profile/entry[@name='$GP_GW_TLS_PROFILE']"
  ```
  + **If your certificate name does not change, you can safely leave these commands  enabled if desired**
  
  
# Prepare the Script for Execution
+ If the cron executes as another logged-in user, you may need to change relative paths to full-paths. (Note – This has been done already above)
  + Ex: “~/.panrc” to “/home/username/.panrc”
  + Or: “~/.cloudflare.ini” to /home/username/.cloudflare.ini
+ Ensure there are no extensions after the filename. In most cases, only underscores and dashes are supported
+ Ensure the cron is executable
  
  ```
  sudo chmod +x /home/psiri/pan_certbot
  ```
  
# Creating a Cronjob for Fully-Automated Certificate Installation
### Note: This article is written for Ubuntu

1. Create a custom cronjob to execute the certificate renewal at your desired interval:
  ```
  sudo vi /etc/cron.d/pan_certbot_cron
  ```
2. Insert the following within the /etc/cron.d/pan_certbot file to ensure the cron generates a log file to debug any issues:
  + The following cronjob will run twice weekly (Midnight Weds and Saturday).  To change execution frequency, modify the schedule to meet your needs.
  + The log file will be timestamped and stored in /var/log/pan_certbot.log
  + **Change “/home/psiri/pan_certbot” to the full path of the script**
  ```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
SHELL=/bin/bash

0 0 * * 3,6 root (/bin/date && /home/psiri/pan_certbot) >> /var/log/pan_certbot.log 2>&1
```

# LetsEncrypt Renewal Limits
LetsEncrypt rate-limits the renewal of certificates by default. It is not uncommon to hit the main limit – 50 certificates per registered domain, per week. Certificate renewals also have a special “Duplicate Certificate” limit of 5/week which you are likely to hit with frequently-running jobs.  

**There is no penalty for exceeding these limits.**  If you have a limited number of certificates to renew, weekly or bi-weekly cronjobs will work perfectly. If you’re running this as a daily or hourly cronjob, you will inevitably see the following error:
```
Renewing an existing certificate
An unexpected error occurred:
There were too many requests of a given type :: Error creating new order :: too many certificates already issued for exact set of domains: *.bitbodyguard.com: see https://letsencrypt.org/docs/rate-limits/
Please see the logfiles in /var/log/letsencrypt for more details.
```

# Validate Your Runs
```
cat /var/log/pan_certbot.log
```
If successful, you should see something similar to the following:
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-cloudflare, Installer None
Renewing an existing certificate
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/bitbodyguard.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/bitbodyguard.com/privkey.pem
   Your cert will expire on 2020-01-21. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4536  100   125  100  4411     34   1204  0:00:03  0:00:03 --:--:--  1204
<response status="success"><result>Successfully imported LetsEncryptWildcard into candidate configuration</result></response> 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4536  100   125  100  4411    103   3667  0:00:01  0:00:01 --:--:--  3666
<response status="success"><result>Successfully imported LetsEncryptWildcard into candidate configuration</result></response> 
set: success [code="20"]: "command succeeded"
set: success [code="20"]: "command succeeded"
commit: success: "Configuration committed successfully"
```

If the cron was successful, but you have it a rate-limit, you will typically see the following:
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-cloudflare, Installer None
Renewing an existing certificate
An unexpected error occurred:
There were too many requests of a given type :: Error creating new order :: too many certificates already issued for exact set of domains: *.bitbodyguard.com: see https://letsencrypt.org/docs/rate-limits/
Please see the logfiles in /var/log/letsencrypt for more details.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4536  100   125  100  4411    101   3582  0:00:01  0:00:01 --:--:--  3583
Successfully imported LetsEncryptWildcard into candidate configuration 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4536  100   125  100  4411    160   5678 --:--:-- --:--:-- --:--:--  5676
Successfully imported LetsEncryptWildcard into candidate configuration 
set: success [code="20"]: "command succeeded"
set: success [code="20"]: "command succeeded"
commit: success: "Configuration committed successfully"
```

You should also be able to check the following locations on the Palo Alto Networks firewall for additional confirmation:
**Monitor –> Logs –> Configuration**
You should see 3-5 operations, depending on whether or not you chose to modify the SSL/TLS service profile(s).  In my setup (PA-850), it takes 7-8 seconds to renew, upload, and commit the configuration (not including actual commit time):

1. A web upload to /config/shared/certificate
2. A web upload to /config/shared/certificate/entry[@name=’LetsEncryptWildcard’]
3. A web “set” command to /config/shared/ssl-tls-service-profile/entry[@name=’GP_PORTAL_PROFILE’]
4. A web “set” command to /config/shared/ssl-tls-service-profile/entry[@name=’GP_EXT_GW_PROFILE’]
5. And a web “commit” operation
