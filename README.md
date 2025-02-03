# play-with-vpn
how to setup vpn with gcp. Credit to https://dhanangw.medium.com/setup-wireguard-vpn-in-google-cloud-platform-67ddb692b2d8. 
great post, however, there is some issue with the client section (dated 20250203). it says `Replace <PRIVATE-IP-OF-WIREGUARD-SERVER> to “10.0.0.1/24”`. at its mobile.conf (client side)

```
[Interface]
PrivateKey = <CLIENT’S-PRIVATE-KEY>
Address = <PRIVATE-IP-OF-WIREGUARD-SERVER>/24
DNS = 1.1.1.1, 1.0.0.1
MTU = 1360
[Peer]
PublicKey = <YOUR-SERVER'S-PUBLIC-KEY>
AllowedIPs = 0.0.0.0/0
Endpoint = <STATIC-IP-OF-GCP-INSTANCE>:51820
```

however it should be 10.0.0.2/24. because when the peer is added to server, it is using 10.0.0.2, as below.

```
sudo wg set wg0 peer <YOUR_CLIENT_PUBLIC_KEY> allowed-ips <YOUR_CLIENT_VPN_IP>
```

then it says: 
```
Replace <YOUR_CLIENT_VPN_IP> to 10.0.0.2/32 Make sure your client already added to server.
sudo wg show wg
```

also, the client/peer of the server, is stored in memory, it is not explicitly shown in the `/etc/wireguard/wg0.conf`. this makes the whole process not declarative.

Now let's begin!



## create a vm on gcp

Create instance at compute engine with below config
* e2-micro
* ubuntu
* default network, enable IP forwarding
* use a reserved static ip, if you want to pay a bit more, but just make sure it has an external ip
then create

## filewall at vpc layer

add filewall rules
* udp:51820 (vpn), use a network tag, then tag the machine. or apply to all machines in the network (a bit overkill).
* tcp:22 (ssh), you dont need to do this, google has default firewall rule to allow tcp:22 for all machines in default network.

I did not try with my own VPC with cloudNAT. but it should work.


## install wireguard on vm


```
sudo apt update && sudo apt upgrade -y
sudo apt install wireguard -y
```


## generate server and client key pairs

here for clients, we use one device per client. if 2 phones using same client, lag is significant.

```
# do the server just once, then record the value
wg genkey | tee server_private.key | wg pubkey > server_public.key

# do below many times and record the valye
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

in the end we will have (suppose i want to do it for me and my wife). save this elsewhere.

```
# server private
SERVER/PRIVATE=
# server public
SERVER/PUBLIC=


# client1 private
CLIENT1/PRIVATE=
# client2 public
CLIENT1/PUBLIC=

# client2 private
CLIENT2/PRIVATE=
# client2 public key
CLIENT2/PUBLIC=

...
```

## now need to find the network config

now to get the `ens4`. well i need to study what it is..

```
root@instance-20250202-093114:/home/yourname# ip -o -4 route show to default | awk '{print $5}'
ens4
```

## finally, the server config

```
vi /etc/wireguard/wg0.conf
```
take note at the `ens4`, and the IP address like `10.0.0.x`.
```
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = SERVER/PRIVATE=
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens4 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens4 -j MASQUERADE

[Peer]
PublicKey = CLIENT1/PUBLIC
AllowedIPs = 10.0.0.2/32

[Peer]
PublicKey = CLIENT2/PUBLIC
AllowedIPs = 10.0.0.3/32
```

turn it on!

```
sudo wg-quick up wg0
```

## filewall at linux layer

```
# Open WireGuard port through firewall
sudo ufw allow 51820/udp
# open port for SSH as well
sudo ufw allow 22/tcp
# Turn on firewall
sudo ufw enable
# Check firewall status, make sure the port for WireGuard and SSH are opened.
sudo ufw status verbose
# Set MTU size to 1360 due to limitation in Google Cloud Platform.
sudo ip link set dev wg0 mtu 1360
```

## the clients...

here pls take note of the `10.0.0.3/24` need to map to the peer of the server side settings!

each client will use unique address, but same endpoint, cos I want to only use one VM on gcp for multiple person. `8.8.8.8` added to DNS. 

Each client needs its own conf because the Address, private key is different. Here I use client2 as an example.

```
vi client2.conf
```
the content. pay attention to the `Address`
```
[Interface]
PrivateKey = CLIENT1/PRIVATE=
Address = 10.0.0.3/24
DNS = 1.1.1.1, 8.8.8.8
MTU = 1360
[Peer]
PublicKey = SERVER/PUBLIC=
AllowedIPs = 0.0.0.0/0
Endpoint = 35.198.200.65:51820
```

to get the qr code

```
sudo apt install qrencode
qrencode -t ansiutf8 < client2.conf
```

now download the wireguard app on your phone, scan the QR code. tada!

## some more important commands
```
sudo systemctl restart wg-quick@wg0
sudo systemctl enable wg-quick@wg0
sudo systemctl status wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg show wg0
```

## things to avoid

below is imparative, lets avoid it..
```
sudo wg set wg0 peer <YOUR_CLIENT_PUBLIC_KEY> allowed-ips <YOUR_CLIENT_VPN_IP>
```

To make it extra safe, avoid port 51820.
