# BIND
BIND DNS Server Configuration on RHEL 8

## What’s An Authoritative DNS Server?
If you own a domain name and want your own DNS server to handle name resolution for your domain name instead of using your domain registrar’s DNS server, then you will need to set up an authoritative DNS server.

An authoritative DNS server is used by domain name owners to store DNS records. It provides authoritative answers to DNS resolvers (like 8.8.8.8 or 1.1.1.1), which query DNS records on behalf of end-users on PC, smartphone, or tablet.

## About BIND
BIND (Berkeley Internet Name Domain) is an open-source, flexible and full-featured DNS software widely used on Unix/Linux due to its stability and high quality. It’s originally developed by UC Berkeley, and later in 1994, its development was moved to Internet Systems Consortium, Inc (ISC).

BIND can act as an authoritative DNS server for a zone and a DNS resolver at the same time. A DNS resolver can also be called a recursive name server because it performs recursive DNS lookups for end users. However, taking two roles at the same time isn’t advantageous. It’s a good practice to separate the two roles on two different hosts.

## Prerequisites
Server needs only 512MB RAM

Please note that you need root privilege when installing software. You can add sudo at the beginning of a command, or use su - command to switch to the root user.

## Set up Authoritative DNS Server on CentOS 8/RHEL 8 with BIND9
You need to run commands in this section on the server.

```
sudo yum update
sudo yum install bind bind-utils -y
```
Check the version information.
```
named -v
```
Start BIND 9
```
sudo systemctl start named
```

Start BIND 9 at boot time
```
sudo systemctl enable named
```
Check status
```
systemctl status named
```
By default, the BIND9 server on CentOS 8/RHEL 8 listens on localhost only. To provide authoritiave DNS service to resolvers on the public Internet, we need to configure it listen on the public IP address. Edit the BIND main configuration file /etc/named.conf with a command-line text editor like vim.
```
sudo yum install vim
vim /etc/named.conf
```

Configure /etc/named.conf for domain example.com and create forward and reverse zones for your domain.
```
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
	listen-on port 53 { 127.0.0.1; <DNS-Server-IP>; };
	listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	secroots-file	"/var/named/data/named.secroots";
	recursing-file	"/var/named/data/named.recursing";
	allow-query     { localhost; 192.168.56.0/24; };

	/* 
	 - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
	 - If you are building a RECURSIVE (caching) DNS server, you need to enable 
	   recursion. 
	 - If your recursive DNS server has a public IP address, you MUST enable access 
	   control to limit queries to your legitimate users. Failing to do so will
	   cause your server to become part of large scale DNS amplification 
	   attacks. Implementing BCP38 within your network would greatly
	   reduce such attack surface 
	*/
	recursion no;

	// enable query log
	querylog yes;

	dnssec-enable yes;
	dnssec-validation yes;

	managed-keys-directory "/var/named/dynamic";

	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";

	/* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
	include "/etc/crypto-policies/back-ends/bind.config";
};

zone "." IN {
	type hint;
	file "named.ca";
};

// forward zone
zone "example.com" IN {
	type master;
	file "fw.example.com";
	allow-update { none; };
	allow-query { any; };
};

// reverse zone
zone "<DNS-Server-IP-without-host>.in-addr.arpa" IN {
	type master;
	file "rev.example.com";
	allow-update { none; };
	allow-query { any; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```
Where `<DNS-Server-IP-without-host>`, for example an ip of 192.168.56.1 -> 56.168.192

In the above configuration, we created a new zone with the zone clause and we specified that this is the master zone. The forward zone file is /var/named/fw.example.com and reverse zone file is /var/namedd/rev.example.com where we will add DNS records. This zone allows query from custom IP range. The allow-query directive in this zone will override the global allow-query directive.

Instead of creating a zone file from scratch, we can use a zone template file. Copy the contents of named.localhost and named.loopback to new files.
```
sudo cp /var/named/named.localhost /var/named/fw.example.com
sudo cp /var/named/named.loopback /var/named/rev.example.com
```
A zone file can contain 3 types of entries:

* Comments: start with a semicolon (;)
* Directives: start with a dollar sign ($)
* Resource Records: aka DNS records

A zone file typically consists of the following types of DNS records.

* The SOA (Start of Authority) record: defines the key characteristics of a zone. It’s the first DNS record in the zone file and is mandatory.
* NS (Name Server) record: specifies which servers are used to store DNS records and answer DNS queries for a domain name. There must be at least two NS records in a zone file.
* MX (Mail Exchanger) record: specifies which hosts are responsible for email delivery for a domain name.
* A (Address) record: Converts DNS names into IPv4 addresses.
* AAAA (Quad A) record: Converts DNS names into IPv6 addresses.
* CNAME record (Canonical Name): It’s used to create alias for a DNS name.
* TXT record: SPF, DKIM, DMARC, etc.

Edit the forward zone file /var/named/fw.example.com
```
$TTL 1D
@	IN SOA	bind53.example.com. hostmaster.example.com. (
					2023042600	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum
; Name Server for this domain
@	IN	NS	bind53.example.com.

; A records
bind53		IN	A	<DNS-Server-IP>
client1	    IN	A	<Client-Machine-IP>
client2	    IN	A	<Client-Machine-IP>
```
Where

* The $TTL directive defines the default Time to Live value for the zone, which is the time a DNS record can be cached on a DNS resolver. This directive is mandatory.
* The $ORIGIN directive defines the base domain.
* Domain names must end with a dot (.), which is the root domain. When a domain name ends with a dot, it is a fully qualified domain name (FQDN).
* The @ symbol references to the base domain.
* IN is the DNS class. It stands for Internet. Other DNS classes exist but are rarely used.

The first record in a zone file is the SOA (Start of Authority) record. This record contains the following information:

* The master DNS server.
* Email address of the zone administrator. RFC 2142 recommends the email address hostmaster@example.com. In the zone file, this email address takes this form: hostmaster.example.com because the @ symbol has special meaning in zone file.
* Zone serial number. The serial number is a way of tracking changes in zone by the slave DNS server. By convention, the serial number takes a date format: yyyymmddss, where yyyy is the four-digit year number, mm is the month, dd is the day, and ss is the sequence number for the day. You must update the serial number when changes are made to the zone file.
* Refresh value. When the refresh value is reached, the slave DNS server will try to read of the SOA record from the master DNS server. If the serial number becomes higher, a zone transfer is initiated.
* Retry value. Defines the retry interval if the slave DNS server fails to connect to the master DNS server.
* Expiry: If the slave DNS server has been failing to make contact with master DNS server for this amount of time, the slave will stop responding to DNS queries for this zone.
* Negative cache TTL: Defines the time to live value of DNS responses for non-existent DNS names (NXDOMAIN).
* TXT records are usually enclosed in double quotes. If you add DKIM record, you also need to enclose the value with parentheses.

Edit the reverse zone file /var/named/rev.example.com
```
$TTL 1D
@	IN SOA	bind53.example.com. hostmaster.example.com. (
					2023042600	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; miniimum
; Name Server for this domain
@	IN	NS	bind53.example.com.
bind53	IN	A	<DNS-Server-IP>

; reverse lookup for Name Server
102	IN 	PTR	bind53.example.com.

; PTR records IP Address to Hostname
101	IN 	PTR	client1.example.com.
103	IN 	PTR	client2.example.com.
```
Then run the following command to check if there are syntax errors in the main configuration file. A silent output indicates no errors are found.
```
sudo named-checkconf
```
Then check the syntax of zone files.
```
sudo named-checkzone example.com /var/named/fw.example.com
```
If there are syntax errors in the zone file, you need to fix it, or this zone won’t be loaded. The following message indicates there are no syntax errors.
```
zone example.com/IN: loaded serial 2023042600
OK
```
Then restart BIND9
```
sudo systemctl restart named
```
Allow firewall for DNS service on port 53
```
firewall-cmd  --add-service=dns --zone=public  --permanent
firewall-cmd --reload

```
Edit /etc/resolv.conf file from a client machine to resolve dns queries using the DNS server
```
# devops.internal NS
search devops.internal
nameserver	<DNS-Server-IP>
```
Use the dig utility to check the NS record of your domain name.
```
dig bind53.example.com
```

### Logging configuration

```
cd /var/log
mkdir bind
chown named:named bind
```

Edit /etc/named.conf and add logging configuration at the end
```logging {
        channel transfers {
            file "/var/log/bind/transfers" versions 3 size 10M;
            print-time yes;
            severity info;
        };
        channel notify {
            file "/var/log/bind/notify" versions 3 size 10M;
            print-time yes;
            severity info;
        };
        channel dnssec {
            file "/var/log/bind/dnssec" versions 3 size 10M;
            print-time yes;
            severity info;
        };
        channel query {
            file "/var/log/bind/query" versions 5 size 10M;
            print-time yes;
            severity info;
        };
        channel general {
            file "/var/log/bind/general" versions 3 size 10M;
        print-time yes;
        severity info;
        };
    channel slog {
        syslog security;
        severity info;
    };
        category xfer-out { transfers; slog; };
        category xfer-in { transfers; slog; };
        category notify { notify; };

        category lame-servers { general; };
        category config { general; };
        category default { general; };
        category security { general; slog; };
        category dnssec { dnssec; };

        category queries { query; };
};

```