---
layout:     post
title:      My home server
date:       2026-08-20 11:21:29
summary:    This is how I config my homeserver
---

I want to keep my server simple so that I don't spend too much time debugging stuff. So there are only a few services:
- VPN: Wireguard
- Reverse proxy with self signed certificates for HTTPS
- Some other services like media, photos backup...

### Wireguard configuration

Why Wireguard? Many says because it's fast, light... I chose it because simply because I think it's "kinda" hard to begin with and hope that I learned new things after doing this so let's dive in

First we need to generate the key for the server side and the client side. Yes you need to do it 2 times, 1 on your server, 1 on your client device, or both on 1 machine but don't forget to change the file's name

```
wg genkey | tee privkey | wg pubkey > pubkey
```

Here we start to write the configuration file on the server side, i will name it **wg0.conf**. In the config file, there are 2 sections _Interface_ to do some networking, define the adress of the server,... while on VPN and _Peer_ to allow the client to connect

    
    [Interface]
    Address = 10.0.0.1/32
    PostUp = iptables -I DOCKER-USER -i wg0 -j ACCEPT; iptables -I DOCKER-USER -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o enp2s0 -j MASQUERADE
    PreDown = iptables -D DOCKER-USER -i wg0 -j ACCEPT; iptables -D DOCKER-USER -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o enp2s0 -j MASQUERADE
    ListenPort = The listen port you want 
    PrivateKey = Put the private key you just generated here
    
    [Peer]
    PublicKey = Public key of the client
    AllowedIPs = 10.0.0.2/32
            

The _PostUp_ and _PreDown_ sections are to bypass the Docker firewall because I use some services running on Docker on my server and also to share the internet/network connection with the VPN client. That's why we can still have access to Internet on the VPN. Dont forget to change _enp2s0_ to your own network interface

And now on the client side, you will need a dynamic DNS domain always point to the server's public IP. I use a free domain on [DuckDNS](https://www.duckdns.org){:target="_blank"}, you can check it out
    
    [Interface]
    Address = 10.0.0.2/32
    PrivateKey = your private key
    
    [Peer]
    PublicKey = the public key of the server
    AllowedIPs = 0.0.0.0/0
    Endpoint = duckdns_domain:the_port_you_want_to_use_for_wireguard
            

And now to we need to enable this Wireguard network interface on both side
```bash
wg-quick up wg0 ##wg0 like the name of the configuration file we wrote
```
To check if it works, we can type this to check if we can see the wg0 interface, then ping to your server from a different network to make sure it works.
```bash
sudo wg
```
Some problems that I got that maybe you can avoid
- Forget to change rules of the firewall to allow traffic
```bash
sudo firewall-cmd --permanent --add-port=your_wireguard_port/udp
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload
```
- Can connect to Internet on VPN but can't access the service web (that runs on Docker container), we need to allow the incoming traffic from WireGuard VPN to reach Docker containers
```bash
sudo iptables -I DOCKER-USER -i wg0 -j ACCEPT
sudo iptables -I DOCKER-USER -o wg0 -j ACCEPT
```
We can add these rules to the `wg0.conf`. In fact, I already put it there
### Self-signed certificates
#### Setting up CA (Certificate Authority)
- Generate a key for our CA, here is our server because this is for self-signed certificate.
```bash
openssl genrsa -aes256 -out ca-key.pem 4096
```
- Create a public certificate for our CA. The `-x509` flag tells OpenSSL to output a self-signed certificate instead of a Certificate Signing Request (CSR)
```bash
openssl req -new -x509 -sha256 -days 365 -key ca-key.pem -out ca.pem
```
#### Create certificate for our service
- First, we need a key
```bash
openssl genrsa -out cert-key.pem 4096
```
- Then we will create a CSR for our service
```bash
openssl req -sha256 -new -key cert-key.pem -out cert.csr
```
- Then we need a `extfile.cnf` with all the alternative names for our services (domain, subdomain)
```
subjectAltName = DNS:yourdomain.com,DNS:www.yourdomain.com,IP:192.168.1.1
```
- Now we can create a self-signed certificate for our service, for Apple devices we should only generate certificates with validity less than 300 days or else it won't be accepted by the device
```bash
openssl x509 -req -sha256 -days 365 -in cert.csr -CA ca.pem -CAkey ca-key.pem -out cert.pem -extfile extfile.cnf -CAcreateserial
```
### Nginx cofig
Create a directory to store the certificates
```bash
sudo mkdir -p etc/nginx/certs
sudo cp cert.pem etc/nginx/certs/
sudo cp cert-key.pem etc/nginx/certs/
```
We will create a snippet to tell the Nginx server to use these certificates, you can also put this directly in the Nginx server config file
```bash
sudo nvim etc/nginx/snippets/self-signed.conf
```
Add the following content
```conf
ssl_certificate /etc/nginx/certs/cert.pem;
ssl_certificate_key /etc/nginx/certs/cert-key.pem;
```
We need another snippet, you can also put this directly in the Nginx server config file
```bash
sudo nvim etc/nginx/snippts/ssl-params.conf
```
And add the following content
```conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
ssl_ecdh_curve secp384r1;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling off;
ssl_stapling_verify off;
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
```
Now we will write the Nginx server config file
```bash
sudo nvim etc/nginx/sites-available/homesv.lan
```
And the content of this file is 

```conf
server { 
	listen 80;
	server_name PUT_YOUR_DOMAIN_HERE;

	return 307 https://$server_name$request_uri;
}

server {
	listen 443 ssl;
	http2 on;
	server_name PUT_YOUR_DOMAIN_HERE;

	include snippets/self-signed.conf;
	include snippets/ssl-params.conf;

	location / {
		proxy_pass http://localhost:PUT_THE_PORT_OF_YOUR_SERVICE;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;

		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
	}
}
```
To enable it move this file to `etc/nginx/sites-enabled` directory, then verify with `sudo nginx -t` then enable it by reload Nginx `sudo systemctl reload nginx`

Now if we try to access the domain then we will see that it is accessible over HTTPS but the browser will warn that the certificate is not trusted, so we need to import it to our device



