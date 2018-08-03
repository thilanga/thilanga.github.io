---
layout: post
title: Linux-Windows Single Sign-On
date: 2009-03-17
comments: true
tags: single-sign-on linux
---

Here I'm going to post very useful Linux-Windows Single Sign-On configuration, for who wants to authenticate their
linux Machines over the Active Directory.

####Setup and Configure Winbind

 * Name Service Switch (NSS):
 >This is a set of capabilities built into the Linux C libraries that allow an application to select a source to validate authentication credentials.

 * Pluggable Authentication Modules (PAM):
 >This extends the standard Unix password authentication mechanism to include central authentication databases such as LDAP, Kerberos, AD and so on.

 * Winbind with Samba:
 >The winbind service uses Samba for configuration information. For AD interoperability, make sure your system is running a current version of Samba (3.05 or newer).

 * Kerberos:
 >Winbind uses Kerberos to get tickets for accessing AD. A Windows domain controller acts as the Key Distribution Center (KDC).

To configure winbind in Centos, launch the **Authentication Configuration** (as superuser (root). `System-config-authentication`
but it doesn't make all the required configuration settings.

Check the Use Winbind option In the **Winbind Settings** window, set the *Security Model* to *ads* and fill in the *Winbind Domain*,
*Winbind ADS Realm* and *Winbind Domain Controllers*. See sample settings below.

```bash
    Domain: flat (NetBIOS) name for the domain (COMPANY)
    Security Model: ADS
    ADS Realm: FQDN for the domain (company.com)
    Domain Controllers: Fully Qualified Domain Name (FQDN) for a domain controller (dc1.company.com)
    Template Shell: /bin/bash
```

>There's a Join Domain option, but don't select it. It might not work, and you won't get sufficient feedback to help resolve problems.
For now, just click OK to save the changes you just entered.

####NOTE:

>To ensure the success of the Active Directory integration, make sure that your Active Directory DNS is working,
you are using the Active Directory DNS, you can ping the domain controllers and that the difference between the domain controllers’ clock is not more than five minutes.


When the authconfig window closes, the console window should show that winbind starts. If this fails, try starting the service manually with the following command:

```bash
    /etc/init.d/winbind start
```

If winbind starts, it will appear on a ps process list like this:

```bash
    # ps -A | grep winbind<br/>3132 ? 00:00:00 winbindd<br/>3133 ? 00:00:00 winbindd
```

####Configuration Files

Authconfig makes changes to three configuration files.

 * Name Service Switch (NSS):
 >This is a set of capabilities built into the Linux C libraries that allow an application to select a source to validate authentication credentials.

 * nsswitch (/etc/nsswitch.conf)
 >The critical entries are passwd and group. Other Linux flavors don't bother assigning winbind to other services. comments and irrelevant information removed

```bash
    passwd: files winbind
    shadow: files winbind
    group: files winbind
    hosts: files dns
    bootparams: nisplus [NOTFOUND=return] files
    ethers: files
    netmasks: files
    networks: files
    protocols: files
    rpc: files
    services: files
    netgroup: files
    publickey: nisplus
    automount: files
    aliases: files nisplus
```

 * system-auth (/etc/pam.d/system-auth):
 >PAM uses a stackable authentication scheme, and each element in the stack must be separately configured.


```bash
    #%PAM-1.0
    # This file is auto-generated.
    # User changes will be destroyed the next time authconfig is run.
    auth required /lib/security/$ISA/pam_env.so
    auth sufficient /lib/security/$ISA/pam_unix.so likeauth nullok
    auth sufficient /lib/security/$ISA/pam_winbind.so use_first_pass
    auth required /lib/security/$ISA/pam_deny.so
    account sufficient /lib/security/$ISA/pam_succeed_if.so uid <>
    account required /lib/security/$ISA/pam_unix.so
    account [default=bad success=ok user_unknown=ignore] /lib/security/$ISA/pam_winbind.so
    password requisite /lib/security/$ISA/pam_cracklib.so retry=3
    password sufficient /lib/security/$ISA/pam_unix.so nullok use_authtok md5 shadow
    password sufficient /lib/security/$ISA/pam_winbind.so use_authtok
    password required /lib/security/$ISA/pam_deny.so
    session required /lib/security/$ISA/pam_limits.so
    session required /lib/security/$ISA/pam_unix.so
```

 * smb.conf (/etc/samba/smb.conf):
 >The idmap entries are important because winbind uses them to maintain a correspondence between AD account names and the User IDs and Group IDs used by Linux.
 Fedora assigns a large range of potential IDs. Typically, other Linux flavors assign a range of 10000-20000.

```bash
    [global]
    workgroup = COMPANY
    server string = Samba Server
    printcap name = /etc/printcap
    load printers = yes
    log file = /var/log/samba/%m.log
    max log size = 50
    security = ads
    socket options = TCP_NODELAY SO_RCVBUF=8192 SO_SNDBUF=8192
    dns proxy = no
    idmap uid = 16777216-33554431
    idmap gid = 16777216-33554431
    template shell = /bin/bash
    winbind use default domain = yes
    password server = dc1.company.com
    realm = COMPANY.COM
 ```

Also, in smb.conf, note the home directory path inserted by authconfig, **/home/%D/%U**. A user, call him winuser1, from an AD domain,
call it company.com, would get a home directory path of **/home/COMPANY/ winuser**. Authconfig does **not** create the domain folder under **/home**.
You must create it manually.

 * krb5 (/etc/krb5.conf):
 >Krb5 issue tickets for authenticat again AD

```bash
    [logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log

    [libdefaults]
    default_realm = COMPANY.COM
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 24h
    forwardable = yes

    [realms]
    DC1.COMPANY.COM = {
    kdc = kerberos.DC1.COMPANY.COM :88
    kdc = kerberos.DC1.COMPANY.COM
    admin_server = kerberos.DC1.COMPANY.COM:749
    default_domain = DC1.COMPANY.COM
    }

    COMPANY.COM = {
    kdc = DC1.COMPANY.COM
    }

    [domain_realm]
    .dc1.Company.com = DC1.COMPANY.COM
    dc1.Company.com = DC1.COMPANY.COM

    dc1.company.com = DC1.COMPANY.COM
    .dc1.company.com = DC1.COMPANY.COM
    [kdc]
    profile = /var/kerberos/krb5kdc/kdc.conf

    [appdefaults]
    pam = {
    debug = false
    ticket_lifetime = 36000
    renew_lifetime = 36000
    forwardable = true
    krb4_convert = false
    }
```

####Joining an AD Domain

You can now join the Linux workstation to the AD domain using the Linux net command. Here's the syntax,
with everything after the first line generated by **net**:

```bash
    # net ads join -U administrator
    administrator's password:
    Using short domain name -- COMPANY
    Joined `Your Machine Name` to realm `COMPANY.COM`
```

####Configure PAM
At this point, a Windows user trying to authenticate at the Linux desktop would get a series of errors because a local home directory isn't present.
as PAM module—mkhomedir automatically creates the home directory Let's include that. To include this module as part of the login process,
change two configuration files under **/etc/pam.d** :

 * login (/etc/pam.d):
 >This file controls authentication from a console prompt.

```bash
    #%PAM-1.0
    auth required pam_securetty.so
    auth required pam_stack.so service=system-auth
    auth required pam_nologin.so
    account required pam_stack.so service=system-auth
    password required pam_stack.so service=system-auth
    session required pam_selinux.so multiple
    session required pam_stack.so service=system-auth
    session optional pam_console.so
    session required pam_mkhomedir.so skel=/etc/skel/ umask=0077
 ```

 * gdm (/etc/pam.d/gdm):
 >This file controls login from a graphical screen.

```bash
    #%PAM-1.0
    auth required pam_env.so
    auth required pam_stack.so service=system-auth
    auth required pam_nologin.so
    account required pam_stack.so service=system-auth
    password required pam_stack.so service=system-auth
    session required pam_stack.so service=system-auth
    session optional pam_console.so
    session required pam_mkhomedir.so skel=/etc/skel/ umask=0077
```

After changing the PAM files, restart the desktop. This is a quick way to ensure that authconfig made the correct boot settings for the required services.
Thats all, now you can login in to your machine with AD login information.

###Comments
>Anonymous
>>this post is very usefull thx!