# play-with-vpn
how to setup vpn with gcp


## create a vm on gcp

Create instance (e2-micro/ubuntu/default network, enable IP forwarding, use a reserved static ip, if you want to pay a bit more, but just make sure it has an external ip, then create)
now we need to add filewall rules to allow udp:51820 and tcp:22. so vpn and ssh can pass


commands

```
sudo apt update && sudo apt upgrade -y

sudo apt install wireguard -y

# get all the keys
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key


```


server_private.key 
SERVER/PRIVATE=
server_public.key 
SERVER/PUBLIC=


client1 private
CLIENT1/PRIVATE=
client2 public
CLIENT1/PUBLIC=

client2 private
CLIENT2/PRIVATE=
client2 public key
CLIENT2/PUBLIC=



now to get the `ens4` 

root@instance-20250202-093114:/home/yourname# ip -o -4 route show to default | awk '{print $5}'
ens4


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

## filewall settings

Open WireGuard port through firewall
sudo ufw allow 51820/udp
open port for SSH as well
sudo ufw allow 22/tcp
Turn on firewall
sudo ufw enable
Check firewall status, make sure the port for WireGuard and SSH are opened.
sudo ufw status verbose
Set MTU size to 1360 due to limitation in Google Cloud Platform.
sudo ip link set dev wg0 mtu 1360

## the clients...

here pls take note of the 10.0.0.3.. need to map to the peer of the server side settings!

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
sudo apt install qrencode
qrencode -t ansiutf8 < above.conf

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
