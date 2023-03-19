# VPS from Scratch

## Introduction

This manual describes way to setup bind as DNS with godaddy, SSL certificate from certbot.
The manual is written for `Ubuntu 20.4`. You will have to replace your server info in configs below.

Replace `<Your server ip address>` with ip address(eg. 10.4.60.1) of your VPS server and `<Your domain name>` with your domain name(eg. piyushxcoder.in).

### Setting up Bind DNS with godaddy

#### Install bind

```
sudo apt install bind9 bind9utils bind9-doc
```

#### Modify `/etc/default/named`

```
OPTIONS="-u bind -4"
```

#### Configure `/etc/bind/named.conf.options`

```
options {
        version "Secured DNS server";

        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation auto;

        //listen-on-v6 { any; };

        allow-query {
                localhost;
                any;
        };

        listen-on port 53 {
                <Your server ip address>;
                localhost;
        };  // listen on private network only

        server-id none;
        allow-transfer { none; };      # disable zone transfers by default
};
```

#### Configure `/etc/bind/named.conf.local`

Add Zone for every domain you are going to use.

```
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

include "/etc/bind/named.conf.certbot";

zone "<Your domain name>" {
        type master;
        file "/etc/bind/db.<Your domain name>";
        allow-transfer { <Your server ip address>; };
        also-notify { <Your server ip address>; };
};
```

#### Create zone file as mentioned in `named.conf.local`

Example Zone file `db.<Your domain name>`

```
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ns1.<Your domain name>. admin.<Your domain name>. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      <Your domain name>.
@       IN      A       <Your server ip address>

        IN      NS      ns1.<Your domain name>.
        IN      NS      ns2.<Your domain name>.
ns1     IN      A       <Your server ip address>
ns2     IN      A       <Your server ip address>

# To redirect www handle it with ngnix
# www	IN	CNAME	<Your server ip address>.

# For Certbot
# _acme-challenge IN NS <Your server ip address>.
```

#### Check Zone files and configuration 
```
sudo named-checkconf
```

#### Restart bind server 
```
sudo service bind9 restart
```

#### Add custom host names with ns1 ns2 subdomain and pointing to your ip addresses as specified in ["Add my custom host names"](https://in.godaddy.com/help/dd-my-custom-host-names-12320).

There after change nameservers for domain with `ns1.<Your domain name>` and `ns2.<Your domain name>`

Do it for every domain you want to point to your DNS

__Note:__ To check if dns is working properly or not you may use `dig @ns1.<Your domain name> <Your domain name>`. It might be also helpful to trace route of dns from root server to yours.

#### References
#### [An Introduction to DNS Terminology, Components, and Concepts](https://www.digitalocean.com/community/tutorials/an-introduction-to-dns-terminology-components-and-concepts)
#### [How To Configure Bind as an Authoritative-Only DNS Server on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-an-authoritative-only-dns-server-on-ubuntu-14-04)

### Setting up Certbot with Bind
#### Install certbot

```
sudo apt install certbot python3-certbot-dns-rfc2136
```

#### Generate a key to secure the update process

```
sudo sh -c "tsig-keygen -a HMAC-SHA512 tsig-key > /etc/bind/tsig.key"
```

#### Create ```/etc/bind/named.conf.certbot```

```
key "tsig-key" {
        algorithm  "hmac-sha512";
        secret "private key";
};

zone "_acme-challenge.<Your domain name>" {
        type master;
        file "/var/lib/bind/db._acme-challenge.<Your domain name>";
        check-names warn;
        update-policy {
                grant tsig-key name _acme-challenge.<Your domain name>. txt;
        };
};
```

Add private key and _achme-challenge zone for each domain and Change permission and ownership

```
$ sudo chown root:bind /etc/bind/named.conf.certbot
$ sudo chmod 640 /etc/bind/named.conf.certbot
```

#### Create zone file for each domain in `/var/lib/bind`

Example of ```/var/lib/bind/db._acme-challenge.<Your domain name>```
```
$ORIGIN .
$TTL 43200	; 12 hours
_acme-challenge.<Your domain name>	IN SOA <Your domain name>. admin.<Your domain name>. (
				2021010211 ; serial
				28800      ; refresh (8 hours)
				7200       ; retry (2 hours)
				604800     ; expire (1 week)
				86400      ; minimum (1 day)
				)
			NS	<Your domain name>.
$TTL 120	; 2 minutes
			TXT	"<Your server ip address>"
```

Change premission and ownership

```
$ sudo chown root:bind /var/lib/bind/db._acme-challenge.<Your domain name>
$ sudo chmod 664 /var/lib/bind/db._acme-challenge.<Your domain name>
```

#### Uncomment `_acme-challenge IN NS <Your domain name>.` in each Zone file `db.<Your domain name>` in `/etc/bind`

#### Add `include "/etc/bind/named.conf.certbot";` in `/etc/bind/named.local`

#### Restart bind server 
```
sudo systemctl restart bind9
```

#### Testing Dynamic Update
Check configs
```
sudo named-checkconf
```

To add the Entry

```
$ sudo nsupdate -k /etc/bind/tsig.key
> server <Your domain name>
> update add _acme-challenge.<Your domain name> 86400 TXT 192.168.1.1
> send
```

To list the Entry

```
dig @<Your domain name> _acme-challenge.<Your domain name> txt
```
You will see 192.168.1.1 in entries. If not then that is a problem!

To delete the Entry
```
$ sudo nsupdate -k /etc/bind/Kcertbot.+165+?????
> server <Your domain name>
> update delete _acme-challenge.<Your domain name> 86400 TXT 192.168.1.1
> send
```

#### Create ```/etc/letsencrypt/dns_rfc2136_credentials.txt```

```
# Target DNS server
dns_rfc2136_server = <Your server ip address>
# Target DNS port
dns_rfc2136_port = 53
# TSIG key name
dns_rfc2136_name = tsig-key
# TSIG key secret
dns_rfc2136_secret =
# TSIG key algorithm
dns_rfc2136_algorithm = HMAC-SHA512
```
Add private key in secret

#### Generate Certificate

```
sudo /usr/bin/certbot certonly --dns-rfc2136 --dns-rfc2136-credentials /etc/letsencrypt/dns_rfc2136_credentials.txt -d '<Your domain name>'  -d '*.<Your domain name>'
```

#### References
#### [Let's Encrypt Wildcard Certificates with certbot, BIND, apache and exim](https://john.daltons.info/home_server_documentation/lets_encrypt.html#:~:text=When%20asking%20for%20a%20wildcard,accept%20dynamic%20updates%20from%20certbot.&text=%24%20sudo%20dnssec%2Dkeygen%20%2Da,b%20512%20%2Dn%20HOST%20certbot.)
