---
layout: post
title: "NTLM Squid Setup using antivirus"
date: 2011-10-6 09:00:00 +0000
categories: [security, network, infra]
tags: [squid, security, infra]
---

# How to Enable NTLM with Squid (with AD Integration)

Let’s discuss how to enable **NTLM authentication with Squid**. Before jumping into the “how”, we need to understand the components required to make this work.


## Components (Our Recipe)

1. **NTP Client**  
   We need the Linux machine to be time-synced with Active Directory so we don’t encounter strange Kerberos errors (often caused by clock skew).

2. **Kerberos Client**  
   We need to authenticate against Active Directory. Kerberos is the authentication platform for Windows Domain Controllers, and it’s required by **Samba/Winbind** to communicate with AD.

3. **Samba Winbind**  
   We need Winbind to join the Linux machine to AD and retrieve Active Directory users/groups.

4. **Squid Server**  
   This is the proxy server we are enabling NTLM authentication on.

5. **Antivirus with ICAP Capability**  
   We need to protect the gateway from viruses. The antivirus must be ICAP-enabled because Squid will use ICAP as the client protocol to communicate with it.

Now that we know the recipe, let’s start.

---

## 1) NTP Client

Active Directory and the Linux machine hosting Squid **must be time-synced**.  
If AD allows a clock skew of (for example) 5 minutes, and your Linux machine differs by 6 minutes, Kerberos authentication will fail—and we don’t want that 🙂.

Edit `/etc/ntp.conf` and add your NTP servers. It’s best practice to configure three servers:

```conf
server 1.1.1.1 iburst
server 2.2.2.2 iburst
server 3.3.3.3 iburst
```

The `iburst` option enables faster initial sync (often 10–15 seconds instead of 5–9 minutes), which is very useful during setup.

After saving the file, enable and start the service (name may differ depending on distro: `ntp` vs `ntpd`):

```bash
chkconfig ntp on
service ntp start
```

To validate sync and debugging:

* `ntptrace`
* `ntpdc` (`listpeers`, `monlist`, `sysinfo`, `ctlstats`)

---

## 2) Kerberos Client

There are two common Kerberos implementations:

* **MIT Kerberos**
* **Heimdal**

Both are RFC-compliant and open source.

Before configuring, it helps to understand:

1. **KDC (Key Distribution Center)**
   Proposed to reduce key exchange risk. It includes:

   * Authentication Server (AS)
   * Ticket Granting Server (TGS)

   The KDC maintains a database of secret keys for entities (hosts, users, services). These keys are used to prove identity, and the KDC also generates session keys for encrypted communication.

2. **Kerberos Realm**
   The realm represents the domain where that database is stored.

---

### Example `/etc/krb5.conf`

```conf
[libdefaults]
        default_realm = DOMAIN.COM
        clockskew = 3000

[realms]
        DOMAIN.COM = {
                kdc = 10.1.0.74
                default_domain = domain.com
                admin_server = 11.11.11.11
        }

[logging]
        kdc = FILE:/var/log/krb5/krb5kdc.log
        admin_server = FILE:/var/log/krb5/kadmind.log
        default = SYSLOG:NOTICE:DAEMON

[domain_realm]
        .domain.com = DOMAIN.COM

[appdefaults]
        pam = {
                ticket_lifetime = 1d
                renew_lifetime = 1d
                forwardable = true
                proxiable = false
                minimum_uid = 1
        }
```

The `clockskew` should match Active Directory configuration. For PAM options, check the man pages—they’re fairly self-explanatory.

---

### Testing Kerberos

Run:

```bash
kinit "user"
klist
```

Expected output will look like:

```bash
Valid starting     Expires            Service principal
10/04/11 12:52:35  10/04/11 22:52:36  krbtgt/DOMAIN.COM@DOMAIN.COM
        renew until 10/05/11 12:52:35
```

If you get similar output, your Kerberos config is likely correct.

---

## 3) Samba Winbind

Once Winbind is configured successfully, the Linux machine acts as a **domain member**. AD users/groups are mapped to local UIDs/GIDs using RID translation.

RID range sizing depends on how many objects exist in AD. My advice: keep the range large—it won’t harm.

---

### Example `smb.conf`

```conf
[global]
        workgroup = DOMAIN       # domain short name
        netbios name = squid-01
        security = ADS
        realm = KERBEROS.REALM   # DOMAIN.COM

        idmap backend = rid
        password server = 1.1.1.1 # AD server IP
        ldap admin dn = cn=winbind,dc=Users,dc=domain,dc=com

        preferred master = no
        local master = no
        domain master = no
        dns proxy = no

        # Wins should be configured: it will spare you netbios broadcast
        wins server = 2.2.2.2

        idmap uid = 1000-99999
        idmap gid = 1000-99999

        idmap backend = idmap_rid:DOMAIN=15777216-37554431
        idmap config DOMAIN: default = yes
        idmap config DOMAIN: backend = rid
        idmap config DOMAIN: range = 15777216-37554431
        idmap alloc DOMAIN: range = 15777216-37554431

        winbind separator = + # some applications get confused with \
        winbind enum users = yes
        winbind enum groups = yes
        winbind nested groups = Yes
        winbind use default domain = yes
        winbind refresh tickets = yes
        allow trusted domains = no
        encrypt passwords = yes
        map to guest = Bad User

        # Winbind socket tuning
        socket options = TCP_NODELAY SO_RCVBUF=16384 SO_SNDBUF=16384
        log level = 3 passdb:5 auth:3 winbind:3
```

Man pages are extremely helpful—use them.

---

## Required Winbind Steps

### 1) DNS Records

Add A and PTR records in AD DNS, refresh zones on the server side, and flush DNS on the client side.

### 2) Join the Domain and Validate

```bash
net ads join -U winbind
net rpc join -U winbind
```

Test join:

```bash
net ads testjoin -U winbind
```

Validate keytab:

```bash
net ads keytab list -U winbind
```

Check trusted domains:

```bash
net rpc trustdom list -U winbind
```

List users/groups:

```bash
net ads user -U winbind
net ads group -U winbind
```

### 3) Start Winbind

Service name may be `winbind` or `winbindd`:

```bash
chkconfig winbind on
service winbind start
```

Validation commands:

```bash
wbinfo -a winbind%password
wbinfo -D domain
wbinfo -t
wbinfo -i winbind
wbinfo -u
wbinfo -g
```

If all of the above succeed, you’re ready for the next step.

---

## 4) Squid (NTLM Authentication)

Before configuration, verify a few things.

### 4.1 Squid Compilation Options

Squid should be compiled with these features (either install a build that includes them, or compile it yourself):

* `--enable-icap-client` (for ICAP integration)
* `--enable-auth` (basic, negotiate, ntlm)
* `--enable-basic-auth-helpers` (ldap, multi-domain ntlm, etc.)
* `--enable-ntlm-auth-helpers` (fakeauth, no_check, etc.)
* `--enable-external-acl-helpers` (custom LDAP queries as ACL rules)

### 4.2 Winbind Pipe Access

Squid must be able to access the Winbind privileged pipe:

```
/var/lib/samba/winbindd_privileged
```

### 4.3 Test the NTLM Helper

```bash
ntlm_auth --helper-protocol=squid-2.5-basic
```

Provide `username password` and validate that it succeeds.

---

## Squid Configuration (`squid.conf`)

Add these lines **before** your `http_access` rules:

```conf
auth_param ntlm program /usr/bin/ntlm_auth --helper-protocol=squid-2.5-ntlmssp
auth_param ntlm children 10
auth_param ntlm keep_alive off

auth_param basic program /usr/bin/ntlm_auth --helper-protocol=squid-2.5-basic
auth_param basic realm Squid proxy-caching web server
auth_param basic casesensitive off
auth_param basic credentialsttl 8 hours

authenticate_ttl 8 hour
```

Add ACL before `http_access`:

```conf
acl authenticated_users proxy_auth REQUIRED
```

### Optional: Custom LDAP Query Authentication

Example authenticating users by `sAMAccountName`:

```conf
auth_param basic program /usr/sbin/squid_ldap_auth -u cn -b "bind dn" \
-D "DOMAIN\winbind" -w password -f "(sAMAccountName=%s)" 1.1.1.1
```

---

## 5) Antivirus Configuration (ICAP)

First, let’s define ICAP.

**ICAP (Internet Content Adaptation Protocol)** is a lightweight point-to-point protocol similar to HTTP. It extends proxy servers with services like:

* Content filtering
* Virus scanning

When a user requests an HTTP object, it can be:

1. Fetched and scanned by the ICAP server, then returned to the proxy if clean
2. Fetched by the proxy, forwarded to the ICAP server for scanning, then returned to the user if clean

These two modes are:

* **REQMOD** (request modification)
* **RESPMOD** (response modification)

We won’t cover configuring the ICAP server itself, since that depends on the antivirus vendor.

---

### Example ICAP Client Config (Kaspersky for Proxy v5.5)

```conf
icap_enable on
icap_send_client_ip on
icap_service is_kav_req reqmod_precache 0 icap://127.0.0.1:1344/av/reqmod
icap_service is_kav_resp respmod_precache 0 icap://127.0.0.1:1344/av/respmod
adaptation_access is_kav_req allow all
adaptation_access is_kav_resp allow all

icap_preview_enable on

# Added by Kaspersky Anti-Virus installer

adaptation_service_set c1 c2
adaptation_access c1 deny icap_bypass
adaptation_access c1 deny POST
adaptation_access c1 deny url_allowed_dst_domains
adaptation_access c1 deny url_allowed_regex
adaptation_access c1 deny java_jvm
adaptation_access c1 deny apple_itunes
adaptation_access c1 allow all
```

**Note:**
`adaptation_service_set` is used for redundancy. If you have multiple ICAP servers, this becomes very useful.
