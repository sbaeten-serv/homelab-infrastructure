## 1. Install Proxmox VE

### 1.1 Download and Flash ISO

1. Download the official Proxmox VE ISO from the [official Proxmox website](https://www.proxmox.com/en/downloads).

2. Create a bootable USB drive using [Rufus](https://rufus.ie/en/).

3. Boot your server from the USB drive and follow the installation wizard.

### 1.2 Fix Repositories

1. Open the Proxmox Web GUI in your browser.

2. Navigate to **Datacenter** -> **[Your Server Node]** -> **Repositories**.

3. Disable both the **Enterprise** and **Ceph Enterprise** repositories.

4. Add the **No-Subscription** repositories for both Proxmox VE and Ceph.

---

## 2. Configure Remote Access with Tailscale

### 2.1 Install Tailscale on Proxmox

1. Open the Proxmox host shell/console and run the official installation script: 

```
curl -fsSL https://tailscale.com/install.sh | sh

```

2. Start Tailscale and authenticate: `tailscale up`

3. Copy the URL generated in the terminal output and open it in your browser to authorize the server.

### 2.2 Client Setup

1. Install Tailscale on your laptop and log in using the same account.

2. You can now reach any device on your home network directly using its Tailscale IP address.

---

## 3. Host & Hardware Optimizations

### 3.1 Prevent Sleep/Suspend on Lid Close (Laptop)

Reference: https://serverfault.com/questions/1045949/

1. Open the systemd logind configuration file: `nano /etc/systemd/logind.conf`

2. Locate and edit the following lines (remove `#` if commented out):

```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore

```

3. Save the file and restart the logind service: `systemctl restart systemd-logind`

### 3.2 Power Saving with TLP

> **Note:** Needs verification / testing in homelab setup.

1. Install the `tlp` software package: `apt update && apt install -y tlp`

2. Open the configuration file to edit power-saving options: `nano /etc/tlp.conf`

3. Disable unused hardware components like Wi-Fi, Bluetooth, and screen output to optimize power consumption.

### 3.3 Fan Control and Sensors

> **Note:** Needs verification / testing in homelab setup.

1. Install hardware monitoring tools: `apt install -y lm-sensors`

2. Configure sensor detection: `sensors-detect`

3. Target behavior: Configure fan control daemon/script to activate fans only when CPU temperature reaches 60°C.

---

## 4. Set Up Postfix Email System (Brevo Relay)

Reference: https://wiki.futo.org/index.php/Introductionto_a_Self_Managed_Life:_a_13_hour%26_28_minute_presentation_by_FUTO_software#Have_your_server_email_you_when_a_hard_drive_is_dying.

### 4.1 Install Prerequisites

1. Update package lists and install Postfix and SASL modules: `apt update && apt install -y postfix libsasl2-modules`

### 4.2 Configure Main Postfix File

1. Edit `/etc/postfix/main.cf`: `nano /etc/postfix/main.cf`

```plaintext
#main.cf
# See /usr/share/postfix/main.cf.dist for a commented, more complete version
# Debian specific: Specifying a file name will cause the first
# line of that file to be used as the name. The Debian default
# is /etc/mailname.
# myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (Debian/GNU)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 3.6 on
# fresh installs.
compatibility_level = 3.6

# TLS parameters
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level=may

smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level=may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination

# Set your FQDN (critical for email headers) --> change to to name of the system!
myhostname = pbs.home.arpa

# Aliases (for root and postmaster)
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases

# Local domains (mail sent to these will be delivered locally)
mydestination = $myhostname, pve, localhost.localdomain, localhost

# Smart host (Brevo SMTP)
relayhost = [smtp-relay.brevo.com]:587

# Enable TLS for outgoing mail
smtp_use_tls = yes

# Enable SASL authentication for Brevo SMTP
smtp_sasl_auth_enable = yes

# SASL password file (must be created and hashed)
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd

# Security options for SASL
smtp_sasl_security_options = noanonymous

# Only allow plain mechanism (required for Brevo)
smtp_sasl_mechanism_filter = plain
sender_canonical_maps = static:YOUR_EMAIL@example.com

# Network restrictions (only allow local mail)
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128

# Mailbox size limit (0 = no limit)
mailbox_size_limit = 0

# Recipient delimiter (optional)
recipient_delimiter = +

# Only listen on localhost (security)
inet_interfaces = 127.0.0.1
inet_protocols = all  

# Maximum message size (100MB)
message_size_limit = 102400000
```

### 4.3 Setup SASL Password File

1. Create and open the SASL authentication file: `nano /etc/postfix/sasl_passwd`

2. Add the Brevo SMTP credentials: `[smtp-relay.brevo.com]:587 YOUR_BREVO_LOGIN_EMAIL:YOUR_BREVO_API_KEY`

3. Generate the Postfix lookup table and update permissions: `postmap /etc/postfix/sasl_passwd && chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db`

4. Reload Postfix service: `systemctl reload postfix`

### 4.4 Automated Drive Failure Alerts

1. Verify `smartmontools` is installed on your Proxmox server: `apt install -y smartmontools`

2. Enable and check status of `smartd`: `systemctl enable --now smartd`

> **Note:** Once Postfix is configured, `smartd` will automatically use the local mail system to dispatch email alerts whenever hard drive failure or SMART attribute degradation is detected.