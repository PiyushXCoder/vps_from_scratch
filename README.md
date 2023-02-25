# VPS from Scratch

## Introduction

This manual describes way to setup bind as DNS with godaddy,
SSL certificate from certbot.
The manual is written for Ubuntu 20.4 and is written for piyushxcoder.in domain name.
You need to replace piyushxcoder.in with your domain

### Setting up Bind DNS with godaddy

* To install bind you need to run

```sudo apt install bind9 bind9utils bind9-doc```

* Modify ```/etc/default/named```

```OPTIONS="-u bind -4"```

* Configure ```/etc/bind/named.conf.options```

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
                103.190.242.178;
                localhost;
        };  // listen on private network only

        server-id none;
        allow-transfer { none; };      # disable zone transfers by default
};
```
Replace ```103.190.242.178``` with you own server ip


* Configure ```sudo nano /etc/bind/named.conf.local```
```
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

include "/etc/bind/named.conf.certbot";

zone "piyushxcoder.in" {
        type master;
        file "/etc/bind/db.piyushxcoder.in";
        allow-transfer { 103.190.242.178; };
        also-notify { 103.190.242.178; };
};
```

Add Zone for every domain you gonna use.

* Create zone file as mentioned in ```named.conf.local```

Example Zone file ```db.piyushxcoder.in```

```

; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ns1.piyushxcoder.in. admin.piyushxcoder.in. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      piyushxcoder.in.
@       IN      A       103.190.242.178

        IN      NS      ns2
        IN      NS      ns1
ns1     IN      A       103.190.242.178
ns2     IN      A       103.190.242.178
```
* Check Zone files and configuration with ```sudo named-checkconf```
* Restart bind server ```sudo service bind9 restart```
* Add custom host names with ns1 ns2 subdomain and pointing to your ip addresses
as specified in ["Add my custom host names"](https://in.godaddy.com/help/add-my-custom-host-names-12320).
There after change nameservers for domain with ns1.piyushxcoder.in and ns2.piyushxcoder.in

Do it for every domain you want to point to your DNS

__Note:__ To check if dns is workin properly or not you may use ```dig @ns1.piyushxcoder.in blog.piyushxcoder.in```. It might be also helpful to trace route of dns from root server to yours.

#### References
* [An Introduction to DNS Terminology, Components, and Concepts](https://www.digitalocean.com/community/tutorials/an-introduction-to-dns-terminology-components-and-concepts)
* [How To Configure Bind as an Authoritative-Only DNS Server on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-an-authoritative-only-dns-server-on-ubuntu-14-04)

### Setting up Certbot with Bind
* Install certbot

```sudo apt install certbot python3-certbot-dns-rfc2136```

* Generate a key to secure the update process

```sudo sh -c "tsig-keygen -a HMAC-SHA512 tsig-key > /etc/bind/tsig.key"```

* create ```/etc/bind/named.conf.certbot```

```
key "tsig-key" {
        algorithm  "hmac-sha512";
        secret "private key";
};

zone "_acme-challenge.piyushxcoder.in" {
        type master;
        file "/var/lib/bind/db._acme-challenge.piyushxcoder.in";
        check-names warn;
        update-policy {
                grant tsig-key name _acme-challenge.piyushxcoder.in. txt;
        };
};
```

add private key and _achme-challenge zone for each domain. Change permission and ownership

```
$ sudo chown root:bind /etc/bind/named.conf.certbot
$ sudo chmod 640 /etc/bind/named.conf.certbot

```

* Create zone file for each domain at ```/var/lib/bind```

Example of ```/var/lib/bind/db._acme-challenge.piyushxcoder.in```
```
$ORIGIN .
$TTL 43200	; 12 hours
_acme-challenge.piyushxcoder.in	IN SOA piyushxcoder.in. admin.piyushxcoder.in. (
				2021010211 ; serial
				28800      ; refresh (8 hours)
				7200       ; retry (2 hours)
				604800     ; expire (1 week)
				86400      ; minimum (1 day)
				)
			NS	piyushxcoder.in.
$TTL 120	; 2 minutes
			TXT	"103.190.242.178"
```
Change permissikn and ownership
```
$ sudo chown root:bind /var/lib/bind/db._acme-challenge.piyushxcoder.in
$ sudo chmod 664 /var/lib/bind/db._acme-challenge.piyushxcoder.in
```
* Now you need to add ```_acme-challenge IN NS mydomain.com.``` in each domain file in ```/etc/bind```
* There after add ```include "/etc/bind/named.conf.certbot";``` in ```/etc/bind/named.local```
* Restart bind server ```sudo systemctl restart bind9```

* Testing Dynamic Update
Check configs
```
sudo named-checkconf
```

To add Entry

```
$ sudo nsupdate -k /etc/bind/tsig.key
> server piyushxcoder.in
> update add _acme-challenge.piyushxcoder.in 86400 TXT 192.168.1.1
> send
```

To list Entry

```
dig @piyushxcoder.in _acme-challenge.piyushxcoder.in txt
```
You will see 192.168.1.1 in entries

To delete Entry
```
$ sudo nsupdate -k /etc/bind/Kcertbot.+165+?????
> server piyushxcoder.in
> update delete _acme-challenge.piyushxcoder.in 86400 TXT 192.168.1.1
> send
```

* Create ```/etc/letsencrypt/dns_rfc2136_credentials.txt```

```

# Target DNS server
dns_rfc2136_server = 103.190.242.178
# Target DNS port
dns_rfc2136_port = 53
# TSIG key name
dns_rfc2136_name = tsig-key
# TSIG key secret
dns_rfc2136_secret =
# TSIG key algorithm
dns_rfc2136_algorithm = HMAC-SHA512
```
Add private key in secret and replace ip

* Generate Certificate

```sudo /usr/bin/certbot certonly --dns-rfc2136 --dns-rfc2136-credentials /etc/letsencrypt/dns_rfc2136_credentials.txt -d 'piyushxcoder.in'  -d '*.piyushxcoder.in'```


#### References
* [Let's Encrypt Wildcard Certificates with certbot, BIND, apache and exim](https://john.daltons.info/home_server_documentation/lets_encrypt.html#:~:text=When%20asking%20for%20a%20wildcard,accept%20dynamic%20updates%20from%20certbot.&text=%24%20sudo%20dnssec%2Dkeygen%20%2Da,b%20512%20%2Dn%20HOST%20certbot.)
