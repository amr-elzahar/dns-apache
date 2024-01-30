# Configure Apache HTTP Server for Virtual Hosts and DNS Server

The main goal is to configure the Apache HTTP Server to serve two virtual hosts and then configure a DNS server to access the virtual hosts using the hostname from a client.

## Steps:

1. Change your directory to `/etc/httpd/conf.d/`.

2. Create two files: `01-virtual-host.conf` and `02-virtual-host.conf`.

3. Paste this configuration into the first file (`01-virtual-host.conf`):

```apache
<VirtualHost *:80>
    DocumentRoot /srv/httpd/virtual-one
    ServerName www.virtual-one.com
    ServerAlias virtual-one.com
</VirtualHost>

<Directory /srv/httpd>
    Require all granted
</Directory>
```

The Directory directive is necessary to give Apache access to the file, as the default configuration in the main configuraion file `httpd.conf` is to deny access to the entire filesystem.

4. Paste this configuration into the second file (`02-virtual-host.conf`):

```apache
<VirtualHost *:80>
    DocumentRoot /srv/httpd/virtual-tow
    ServerName www.virtual-tow.com
    ServerAlias virtual-tow.com
</VirtualHost>

<Directory /srv/httpd>
    Require all granted
</Directory>
```

5. If you have SELinux in enforcing mode, you need to change the context of the `/srv/httpd` directory for the HTTP server to be able to access it:

```
chcon -R -t httpd_sys_content_t /srv/httpd
```

The preceding command will change the context temporarily. If you want it to persist, run the following command:

```
semanage fcontext -a -t httpd_sys_content_t /srv/httpd
```

6. In the `/srv/httpd` directory for each virtual host, create an `index.html` file and write anything into it.

7. For the DNS server, install the bind, bind-utils, and bind-chroot packages:

```
yum install -y bind bind-utils bind-chroot
```

8. Allow the DNS through the firewall:

```
firewall-cmd --add-service=dns --permanent
firewall-cmd --reload
```

9. Use the bind-chroot service instead of the normal service for better security.

10. To activate the `named-chroot` service, set it up using the `named-chroot-setup` service. This service will run a script located at `/usr/libexec/setup-named-chroot.sh` and mask the normal service `named`.

11. In the `/etc/named.conf` file, write the options block:

```
options {
listen-on port 53 { 127.0.0.1; 172.16.170.128; };
listen-on-v6 port 53 { none; };
directory "/var/named";
dump-file "/var/named/data/cache_dump.db";
statistics-file "/var/named/data/named_stats.txt";
memstatistics-file "/var/named/data/named_mem_stats.txt";
secroots-file "/var/named/data/named.secroots";
recursing-file "/var/named/data/named.recursing";
allow-query { localhost; 172.16.170.0/24; };
allow-recursion { localhost; 172.16.170.0/24; };
}
```

`listen-on`: The NIC the DNS will listen on.

`allow-query`: This option specifies the IP addresses or subnets that are allowed to send queries to this BIND server.

`directory`: This option specifies the directory where BIND will look for its zone files and other configuration data.

`allow-recursion`: This option specifies the IP addresses or subnets that are allowed to use recursion (querying other DNS servers on behalf of clients) on this BIND server.

12. Add the forward zone block for each host in the same file:

```
zone "virtual-one.com" IN {
type master;
file "/var/named/virtual-one.forward";
allow-query { any; };
};

zone "virtual-two.com" IN {
type master;
file "/var/named/virtual-two.forward";
allow-query { any; };
};
```

`file`: Specifies the zone file for the domain name.

Note: If we put a relative path in the file directive, it will be relative to the directory directive in the options block.

13. Create a zone file for each host. For example:

```
$TTL 8h
@ IN SOA ns1.virtual-one.com. hostmaster.virtual-one.com. (
0 ; serial
1d ; refresh
3h ; retry
3d ; expire
3h ) ; minimum

          IN NS   ns1.virtual-one.com.

ns1 IN A 192.16.170.128
www IN A 172.16.170.128
```

`IN NS ns1.virtual-one.com.`: Specifies the name server (NS) for the zone (mandatory).

`ns1 IN A 192.16.170.128`: Defines an address (A) record for the subdomain ns1 (mandatory).

`www IN A 172.16.170.128`: Defines an address (A) record for the subdomain ns1 (mandatory).

14. Set correct permissions on each zone file. For example:

```
chown root:named /var/namedvirtual-one.forward
```

Then restore its SELinux context:

```
restorcon -Rv /var/named/virtual-one.forward
```

15. Restart the `named-chroot` service:

```
systemctl restart named-chroot
```

16. Configure the client to use this DNS server by adjusting the nameserver directive in the `/etc/resolv.conf` file and have it set to the IP of the DNS server and then use the curl command to access the host:

```
curl www.virtual-one.com
```
