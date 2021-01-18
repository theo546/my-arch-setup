# dnscrypt-proxy

Download and install my configuration file that start a DNS server on localhost which use Cloudflare DoH service:
```
wget -O /etc/dnscrypt-proxy/dnscrypt-proxy.toml https://raw.githubusercontent.com/theo546/my-arch-setup/master/dnscrypt-proxy/dnscrypt-proxy.toml
```

*Note:* my configuration was made to work with IPv6 alongside IPv4.  
Set `ipv6_servers` to `false` and remove `cloudflare-ipv6` from the `server_names` array to not use IPv6.

[You can also configurate a static DNS server if the one you want isn't on the DNSCrypt default list.](https://github.com/DNSCrypt/dnscrypt-proxy/wiki/Configuration#an-example-static-server-entry)

Activate and start the `dnscrypt-proxy` service using:
```
systemctl enable dnscrypt-proxy
systemctl start dnscrypt-proxy
```

You can now set `127.0.0.1` as your DNS server in your network settings.

___

You can also enable the local DoH server by generating an SSL certificate and uncommenting the last lines in the `dnscrypt-proxy.toml` configuration file.  
Run this command as root:
```
openssl req -x509 -nodes -newkey rsa:2048 -days 5000 -sha256 -keyout /etc/dnscrypt-proxy/localhost.pem -out /etc/dnscrypt-proxy/localhost.pem
```
A 2048 bits RSA key is more than enough since the connection will be established locally only.

You can now add your DoH local server as `https://127.0.0.1:3000/dns-query`, follow [this link instructions](https://github.com/DNSCrypt/dnscrypt-proxy/wiki/Local-DoH#how-to-enable-esni-in-firefox) to tell Firefox to use your DoH server.