# Postfix configuration
## Sources
* [Install and configure send only Postfix SMTP server on Ubuntu 18.04](https://bogomolov.tech/postfix-self-hosted-smtp/)
* [Chapter 2. Deploying and configuring a Postfix SMTP server](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/deploying_mail_servers/assembly_mail-transport-agent_deploying-mail-servers#proc_installing-and-configuring-postfix_assembly_mail-transport-agent)
* [How do I change postfix sender address?](https://unix.stackexchange.com/questions/379175/how-do-i-change-postfix-sender-address)

## Install 
```bash
apt-get update
apt install mailutils
apt-get install opendkim opendkim-tools
```

## Configuration
1. Open/create `/etc/mailname` set its contents to:
   ```
   eliaseriksson.io
   ```
### Opendkim
1. Add the user to the group
   ```bash
   gpasswd -a postfix opendkim
   ```
1. Create a folder for DKIM keys:
   ```bash
   mkdir -p /etc/opendkim/keys
   chown -R opendkim:opendkim /etc/opendkim
   chmod go-rw /etc/opendkim/keys
   ```
1. Create opendkim signing table:
   ```bash
   touch /etc/opendkim/signing.table
   ```
   with contents:
   ```
   *@eliaseriksson.io    default._domainkey.eliaseriksson.io
   ```
1. Create key table:
   ```bash
   touch /etc/opendkim/signing.table
   ```
   with contents
   ```
   default._domainkey.eliaseriksson.io     eliaseriksson.io:default:/etc/opendkim/keys/eliaseriksson.io/default.private
   ```
1. Create keys folder for domain and generate keys:
   ```bash
   mkdir /etc/opendkim/keys/eliaseriksson.io
   opendkim-genkey -b 2048 -d eliaseriksson.io -D /etc/opendkim/keys/eliaseriksson.io -s default -v
   chown opendkim:opendkim /etc/opendkim/keys/eliaseriksson.io/default.private
   ```
1. Open `/etc/opendkim/trusted.hosts` and set its contents to trusted hosts i.e:
   ```
   127.0.0.1
   localhost
   *.eliaseriksson.io
   192.168.1.20
   192.168.1.
   172.22.215.75
   192.168.1.180
   ```
1. Open `/etc/opendkim.conf` and set:
   ```
   Socket              inet:8892@localhost
   Canonicalization    simple
   Mode                sv
   SubDomains          no
   AutoRestart         yes
   AutoRestartRate     10/1M
   Background          yes
   DNSTimeout          5
   SignatureAlgorithm  rsa-sha256
   UserID              opendkim
   KeyTable            refile:/etc/opendkim/key.table
   SigningTable        refile:/etc/opendkim/signing.table
   ExternalIgnoreList  /etc/opendkim/trusted.hosts
   InternalHosts       /etc/opendkim/trusted.hosts
   ```
1. Reload the service:
   ```bash
   service opendkim restart
   ```

### Postfix
1. Open `/etc/mailutils.conf` and set:
   ```
   address {
     email-domain eliaseriksson.io;
   };
   ```
1. Open `/etc/postfix/main.cf` and set:
   ```
   inet_interfaces = all
   mydomain = eliaseriksson.io
   myorigin = /etc/mailname
   mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.0.0/16
   milter_default_action = accept
   milter_protocol = 2
   smtpd_milters = inet:localhost:8891
   non_smtpd_milters = inet:localhost:8891
   ```
   and unset `myhostname` and `mydestination`
1. Reload the service:
   ```bash
   systemctl restart postfix
   ```
### DNS
* DKIM: Add `TXT` record with name `default._domainkey` and contents:
   ```
   "v=DKIM1; h=sha256; k=rsa; p=DKIM PUBLIC KEY GOES HERE"
   ```
* DMARC: Add `TXT` record with name `_dmarc` with contents:
   ```
   "v=DMARC1; p=none;"
   ```
* SPF: Add or modify a `TXT` record with name `eliaseriksson.io` with contents:
   ```
   "v=spf1 mx a:eliaseriksson.io ~all"
   ```

### Testing
Test by :
* sending mail from with `mail` command:
   ```bash
   echo "This is the body of the email" | mail -r no-reply@eliaseriksson.io -s "This is the subject" receiver@gmail.com
   ```
* Using `swaks` from remote user on local network:
   ```bash
   apt install swaks
   swaks --from no-reply@eliaseriksson.io --to receiver@gmail.com --server 192.168.abc.xyz:587
   ```