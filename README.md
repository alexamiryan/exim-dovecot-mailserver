# Howto setup mailserver using Exim and Dovecot

Imagine we have a domain `mail.example.com` and we want to set it up as mailserver. Point that domain to your server's IP address. For the server we will need CentOS 7.

Install EPEL repo

`yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm`

Install packages

`yum install -y vim htop exim dovecot spamassassin certbot wget cyrus-sasl`

Open `/etc/exim/exim.conf`

Set you hostname

`primary_hostname = mail.example.com`
`domainlist local_domains = @ : localhost : localhost.localdomain : mail.example.com`

```
local_from_check = false
local_sender_retain = true
untrusted_set_sender = true
```

Uncomment this line to allow authentication with system users

`auth_advertise_hosts = ${if eq {$tls_cipher}{}{}{*}}`

Comment this line

`#auth_advertise_hosts = `

Add this in `remote_smtp:` section

```
headers_remove = Received
dkim_domain = mail.example.com
dkim_selector = my_selector
dkim_private_key = /etc/exim/dkim_priv.key
dkim_canon = relaxed
```

Set this as a `local_delivery:` section

```
local_delivery:
  driver = appendfile
  directory = $home/Maildir
  maildir_format
  maildir_use_size_file
  delivery_date_add
  envelope_to_add
  return_path_add
```

And lastly uncomment `PLAIN:` and `LOGIN:` sections at the end of the file

```
PLAIN:
  driver                     = plaintext
  server_set_id              = $auth2
  server_prompts             = :
  server_condition           = ${if saslauthd{{$2}{$3}{smtp}} {1}}
  server_advertise_condition = ${if def:tls_in_cipher }

LOGIN:
  driver                     = plaintext
  server_set_id              = $auth1
  server_prompts             = <| Username: | Password:
  server_condition           = ${if saslauthd{{$1}{$2}{smtp}} {1}}
  server_advertise_condition = ${if def:tls_in_cipher }
```

Open `/etc/dovecot/conf.d/10-master.conf`

Add this section

```
unix_listener auth-client {
    mode = 0660
    user = exim
}

```

Open `/etc/dovecot/conf.d/10-mail.conf`

Set `mail_location`

`mail_location = maildir:~/Maildir`

---

Ok now start and enable services

```
systemctl start saslauthd
systemctl enable saslauthd
systemctl start exim
systemctl enable exim
systemctl start dovecot
systemctl enable dovecot
```

Add some users and set passwords for them

```
useradd contact
passwd contact
```

Change hostname

```
vim /etc/hostname
hostname mail.example.com
```

Generate DKIM keys, save private one in `/etc/exim/dkim_priv.key` and public key in DNS

Enjoy!