# 1. Contexte

L'hôte Proxmox reçoit sa connexion via l'interface wifi. Pour que les VM puisse obtenir l'accès à internet et que la carte wifi ne peut pas se lier au linux bridge, on utilise le NAT forwarding et masquerade mais aussi dnsmasq comme serveur DHCP.

# 2. Configuration du NAT

Dans le fichier /etc/network/interfaces ajoutez les lignes suivantes (à adapter selon votre configuration)

```
# Pont pour vos VM (Réseau privé NAT)
auto vmbr0
iface vmbr0 inet static
        address 10.0.0.0/24
        bridge-ports enp4s0
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s '10.0.0.0/24' -o wlp5s0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '10.0.0.0/24' -o wlp5s0 -j MASQUERADE
```

Explications : 

- address : est l'adresse de votre réseau
- bridge-ports enp4s0 : j'ai lié l'interface ethernet au bridge linux (tout appareil connecté physiquement accèdera au réseau fourni par l'interface)
- bridge-stp off : indique que le spanning tree protocol est désactivé (servant de base à empêcher les tempêtes de broadcast), dans notre situation il n'y a qu'un seul lien vers les périphériques physiques, du coup, le STP est inutile.
- bridge-fd : Bridge forward delay : est le délai de transfert appliqué par le pont réseau. On force le bridge à passer instantanément du mode désactivé au mode de transfert (forwarding)
- post-up echo 1 > /proc/sys/net/ipv4/ip_forward : activation du forwarding d'ip
- post-up iptables -t nat -A POSTROUTING -s '10.0.0.0/24' -o wlp5s0 -j MASQUERADE
   post-down iptables -t nat -D POSTROUTING -s '10.0.0.0/24' -o wlp5s0 -j MASQUERADE : servent à l'activation du NAT via des règles iptable et MASQUERADE

# 3. Configuration du DHCP avec dnsmasq

installation de dnsmasq
```
apt update && apt install -y dnsmasq
```

Configuration de dnsmasq avec un fichier custom (ex : /etc/dnsmasq.d/proxmox-nat.conf) à adapter en fonction de votre situation :
```
nano /etc/dnsmasq.d/proxmox-nat.conf
```

```
# Écouter uniquement sur l'interface virtuelle NAT
interface=vmbr1
bind-interfaces

# Plage DHCP : début, fin, masque, durée du bail
dhcp-range=10.10.10.10,10.10.10.200,255.255.255.0,12h

# Passerelle par défaut transmise aux clients (l'IP de votre vmbr1)
dhcp-option=option:router,10.10.10.1

# Serveurs DNS transmis aux clients (DNS Cloudflare / Google ou votre box)
dhcp-option=option:dns-server,1.1.1.1,8.8.8.8

# Désactiver le service DNS de dnsmasq si vous ne voulez que le DHCP (optionnel)
# port=0
```

On redémarre le service
```
systemctl restart dnsmasq
systemctl enable dnsmasq
```