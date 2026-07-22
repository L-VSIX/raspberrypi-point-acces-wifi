# 📶 Transformation d'un Raspberry Pi en point d'accès Wi-Fi

> Extension du réseau LAN au sans-fil, sans point d'accès du commerce : le Pi agit en pont (bridge) entre `eth0` et `wlan0`.

## Principe

Le SSID `Raidaporter_LAN` diffuse directement le réseau LAN, sans NAT ni sous-réseau additionnel — le Pi se comporte comme un simple commutateur sans-fil transparent.

## Procédure

**1. Installation des paquets**
```bash
sudo apt update
sudo apt install -y vlan
sudo apt install -y hostapd bridge-utils ifupdown
sudo systemctl stop hostapd
```

**2. Charger et persister le module 8021q
```bash
echo "8021q" | sudo tee -a /etc/modules
sudo modprobe 8021q
lsmod | grep 8021q
```
**3. Retrait des interfaces de NetworkManager** (`/etc/NetworkManager/NetworkManager.conf`)
```ini
[keyfile]
unmanaged-devices=interface-name:eth0;interface-name:eth0.30;interface-name:br0;interface-name:wlan0
```
```bash
sudo systemctl restart NetworkManager
```

**4. Déclaration du bridge** (`/etc/network/interfaces`)
```ini
auto lo
iface lo inet loopback

# Interface physique brute (pas d'IP)
auto eth0
iface eth0 inet manual

# Sous-interface VLAN 30 (GES)
auto eth0.30
iface eth0.30 inet manual
    vlan-raw-device eth0

# Bridge liant le VLAN 30 et le WiFi
auto br0
iface br0 inet dhcp
    bridge_ports eth0.30
    bridge_stp off
    bridge_waitport 0
    bridge_fd 0
```

**5. Configuration hostapd**
```ini
interface=wlan0
bridge=br0
driver=nl80211
ssid=Raidaporter_LAN
hw_mode=g
channel=6
wmm_enabled=0
auth_algs=1
wpa=2
wpa_passphrase=<motdepasse>
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

**6. Activation**
```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
sudo reboot
```

Après redémarrage, `ip a show br0` doit afficher l'IP DHCP du LAN (`192.168.36.2`), confirmant que le bridge hérite bien de l'identité réseau du LAN.

## Limite connue

Le Raspberry Pi héberge à la fois le nœud Wi-Fi et le nœud secondaire Tailscale — les deux usages sont aujourd'hui **mutuellement exclusifs** faute d'avoir stabilisé leur cohabitation. C'est une piste d'amélioration identifiée.

## Repos liés

- [`tailscale-routage-sous-reseaux`](https://github.com/L-VSIX/tailscale-routage-sous-reseaux)
- [`opnsense-segmentation-vlan`](https://github.com/L-VSIX/opnsense-segmentation-vlan)

## Auteur

**Lilian Vertueux** — [LinkedIn](https://www.linkedin.com/in/lilian-vertueux/)
