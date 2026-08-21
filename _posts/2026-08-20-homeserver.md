---
layout:     post
title:      My home server
date:       2026-06-08 11:21:29
summary:    This is how I config my homeserver
---

Wireguard configuration
-----------------------

Why Wireguard? Many says because it's fast, light... I chose it because simply because I think it's "kinda" hard to begin with and hope that I learned new things after doing this so let's dive in

First we need to generate the key for the server side and the client side. Yes you need to do it 2 times, 1 on your server, 1 on your client device, or both on 1 machine but don't forget to change the file's name

    wg genkey | tee privkey | wg pubkey > pubkey

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

And now on the client side

    
    [Interface]
    Address = 10.0.0.2/32
    PrivateKey = your private key
    
    [Peer]
    PublicKey = the public key of the server
    AllowedIPs = 0.0.0.0/0
    Endpoint = Public_IP_of_the_server:the_port_you_want_to_use
            

And now to we need to enable this Wireguard network interface on both side

    wg-quick up wg0 ##wg0 like the name of the configuration file we wrote

To check if it works, we can type this to check if we can see the wg0 interface, then use _ping_ to make sure it works.
