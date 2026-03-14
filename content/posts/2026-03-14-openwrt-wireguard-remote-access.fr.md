---
title: "Configurer WireGuard VPN sur OpenWrt (accès LAN, full tunnel, Cloudflare DDNS)"
date: 2026-03-14
lastmod: 2026-03-14
draft: false
translationKey: "openwrt-wireguard-remote-access"

slug: "openwrt-wireguard-vpn-acces-distant-reseau-local-internet"
aliases:
  - "/posts/2026-03-14-openwrt-wireguard-remote-access/"

description: "Guide pas à pas pour configurer WireGuard sur OpenWrt : règles firewall, redirection UDP, Cloudflare DDNS, onboarding Android par QR code, full tunnel et vérifications sécurité."
summary: "Tutoriel pratique OpenWrt 24.10.5 (Linksys MR8300) : configuration wg0, zones firewall, redirection du port 32541, Cloudflare DDNS, client Android, dépannage et durcissement."

author:
  - "Maxime Morel-Bailly"

tags:
  - "wireguard"
  - "openwrt"
  - "vpn"
  - "reseau"
  - "android"
  - "cybersecurite"

keywords:
  - "wireguard openwrt"
  - "vpn openwrt"
  - "openwrt wireguard cloudflare ddns"
  - "wireguard android qr code"
  - "openwrt 24.10.5"
  - "linksys mr8300"
  - "huawei hg8145x6"
  - "port forwarding"
  - "ddns"
---

Quand on pense VPN, on pense souvent d'abord à la confidentialité sur Wi-Fi public ou l'accès à des contenus géo-restreints. Mon besoin principal est un peu différent : **accéder à mon réseau local à distance** et, si nécessaire, **faire sortir tout le trafic Internet de mon téléphone ou de mon portable via ma connexion domestique**.

Concrètement, je veux :

1. accéder à des ressources locales quand je suis hors de chez moi
2. administrer certains services sans les exposer directement sur Internet
3. utiliser ma connexion maison comme point de sortie sur des réseaux tiers
4. garder une architecture simple, maintenable et adaptée au mobile

Pour ça, **WireGuard** est un très bon choix. C'est plus simple à maintenir que beaucoup d'anciens VPN, performant, et très bien intégré dans **OpenWrt**.

Cet article est basé sur mon setup :

- un **Linksys MR8300** avec **OpenWrt 24.10.5**
- le routeur OpenWrt placé derrière une **box FAI Huawei**
- un accès distant principalement depuis **Android**

L'objectif est d'obtenir à la fois :

- l'accès au **LAN derrière OpenWrt**
- un **routage Internet complet** via la maison (`AllowedIPs = 0.0.0.0/0` côté client)

Je ne vois pas WireGuard ici comme une simple option de confort. C'est un composant d'accès distant et d'administration, donc il faut une base réseau/sécurité propre.

Si vous voulez aller directement à l'implémentation, sautez à [l'étape 1](#etape-1-creer-linterface-wireguard-dans-luci).

Ce guide vise un cas de recherche très concret : **OpenWrt + WireGuard derrière box FAI + Android + DDNS + full tunnel**.

## Ce que fait ce setup

Ce montage est pensé pour un réseau domestique où :

- la **box FAI** reste en place
- le **routeur OpenWrt** est derrière
- un **port UDP** est redirigé depuis la box vers OpenWrt
- les clients se connectent en **WireGuard**
- le VPN fournit à la fois **accès LAN** et **full tunnel Internet**

C'est important, car beaucoup de tutoriels supposent un routeur OpenWrt directement exposé à Internet. Ce n'est pas mon cas ici.

## Prérequis pour OpenWrt derrière une box FAI

Avant de commencer, il me faut :

- **OpenWrt 24.10.5** installé
- accès admin à **LuCI OpenWrt** et à la box **Huawei HG8145X6**
- la possibilité de créer une redirection de port **UDP** sur la box FAI
- une IP WAN stable pour OpenWrt côté box (idéalement réservation DHCP)
- une stratégie **DynDNS** si l'IP publique change
- les paquets WireGuard installés sur OpenWrt

Sur mon routeur, j'ai installé :

```bash
opkg update
opkg install kmod-wireguard wireguard-tools luci-proto-wireguard
```

Ces paquets peuvent aussi être installés via **LuCI > System > Software**.

Le paquet `luci-proto-wireguard` est nécessaire si je veux gérer WireGuard et les peers directement dans LuCI.

![LuCI software page showing the WireGuard packages after clicking Update lists and filtering for wireguard](/images/openwrt/openwrt-luci-wireguard-packages.png)

*Page LuCI après **Update lists...**, filtre `wireguard`, avec `luci-proto-wireguard`, `kmod-wireguard` et `wireguard-tools`.*

Détail important : **j'ai dû redémarrer le routeur** après installation pour voir apparaître WireGuard dans la liste des protocoles LuCI.

## Pourquoi ce choix plutôt qu'exposer des services

Si le besoin est d'accéder à un seul service interne, la tentation est de publier un port directement sur la box.

Problème :

- chaque service exposé augmente la surface d'attaque
- le contrôle d'accès devient dispersé
- exposition Internet et administration se mélangent
- les box FAI sont souvent limitées pour gérer ça proprement

Avec un VPN :

- **un seul point d'entrée** est exposé
- l'authentification repose sur des **clés cryptographiques**
- le LAN reste privé
- l'administration distante devient plus cohérente

## Bénéfices et limites

### Ce que ce setup apporte

- accès distant au **LAN**
- sortie Internet via la maison
- configuration légère sur OpenWrt
- bonne compatibilité clients (**Android**, **Linux**, **Windows**, **macOS**, **iOS**)
- surface d'exposition réduite (un port UDP)

### Ce qu'il faut accepter

- gestion du **NAT / port forwarding** côté box FAI
- nécessité d'un **DynDNS** si l'IP publique change
- limitation par l'upload de la connexion maison
- full tunnel plus coûteux en batterie mobile
- risque d'exposition excessive du LAN en cas de mauvaise config

## Architecture (vue d'ensemble)

```text
Android / Autre client
       |
       |  WireGuard (UDP)
       v
[ Internet ]
       |
       v
[ Box FAI Huawei HG8145X6 ]
       |
       |  redirige un port UDP
       v
[ Routeur OpenWrt MR8300 ]
       |
       +--> LAN local
       |
       +--> sortie Internet via le WAN OpenWrt
```

Le point clé : **OpenWrt n'est pas en bordure Internet**. Il est derrière la box FAI. Il faut donc au minimum :

1. une redirection UDP de la box vers l'IP WAN OpenWrt
2. idéalement une réservation DHCP (ou IP fixe WAN OpenWrt) sur la box
3. un moyen stable de retrouver l'IP publique (DynDNS)

## Plan d'adressage recommandé

Exemple simple :

- LAN OpenWrt : `192.168.10.0/24`
- IP LAN OpenWrt : `192.168.10.1`
- Réseau WireGuard : `10.203.0.0/24`
- IP WireGuard routeur : `10.203.0.1`
- Client Android : `10.203.0.2`
- Autre client (optionnel) : `10.203.0.3`

## Etape 1 : créer l'interface WireGuard dans LuCI

Dans **Network > Interfaces**, créez une interface `wg0` en protocole **WireGuard VPN**.

![LuCI interface creation dialog with WireGuard VPN selected as the protocol for wg0](/images/openwrt/openwrt-wg0-create.png)

Après création, LuCI ouvre la page de configuration de `wg0`.

Générez la paire de clés avec le bouton **Generate new key pair**.

![LuCI WireGuard interface showing the Generate new key pair button](/images/openwrt/openwrt-wg0-generate-key-button.png)

![LuCI WireGuard interface after generating a new private/public key pair](/images/openwrt/openwrt-wg0-generated-keypair.png)

Valeurs principales :

- **private key** : générée par OpenWrt
- **listen port** : port UDP haut non standard, par exemple `32541/UDP`
- **interface address** : `10.203.0.1/24`

![Screenshot of main wg0 configuration screen with 10.203.0.1/24 and the chosen listen port.](/images/openwrt/openwrt-wg0-port-and-ip.png)

Dans les réglages avancés, **désactivez `Use default gateway`** pour ce scénario serveur.

![Screenshot of advanced wg0 configuration screen with disable default gateway.](/images/openwrt/openwrt-wg0-disable-default-gateway.png)

Dans l'onglet firewall de `wg0`, on peut d'abord associer à une zone existante ; on créera la zone `vpn` à l'étape 2.

![Screenshot of VPN wg0 configuration screen with default configuration.](/images/openwrt/openwrt-wg0-vpn-zone-default.png)

Puis enregistrez l'interface.

![Screenshot of VPN wg0 configuration saving button.](/images/openwrt/openwrt-wg0-save-interface.png)

## Etape 2 : rattacher l'interface à une zone firewall

Créez une zone dédiée `vpn` :

- `Name`: `vpn`
- `Input`: `accept`
- `Output`: `accept`
- `Intra zone forward`: `reject`
- `Masquerading`: désactivé (NAT reste sur `wan`)
- `Covered networks`: `wg0`
- `Allow forward to destination zones`: `lan` et `wan`
- `Allow forward from source zones`: vide
- Onglet advanced : IPv4 only si votre setup est IPv4 only

![Screenshot of the new OpenWrt vpn zone configuration for WireGuard.](/images/openwrt/openwrt-new-vpn-zone-configuration.png)

Puis sauvegardez.

![Screenshot of VPN wg0 configuration saving button.](/images/openwrt/openwrt-vpnzone-save.png)

## Etape 3 : autoriser le port WireGuard sur OpenWrt

Dans **Firewall > Traffic Rules**, ajoutez :

- `Name`: `Allow-WireGuard-Inbound`
- `Protocol`: `UDP`
- `Source zone`: `wan`
- `Destination`: `This device` (pas `vpn`)
- `Destination port`: `32541`
- `Action`: `Accept input`
- Advanced : `IPv4 only` si setup IPv4 only

![OpenWrt traffic rule allowing inbound WireGuard UDP traffic from WAN to this device.](/images/openwrt/opnwrt-fw-allow-wireguard-inbound-main-settings.png)

## Etape 4 : rediriger le port sur la box FAI

Sur la box FAI, redirigez `UDP/32541` vers l'IP WAN OpenWrt (exemple `192.168.3.8`).

- `Protocol`: `UDP`
- `External port`: `32541-32541`
- `Internal port`: `32541-32541`
- `External source IP`: vide (sauf besoin de restriction)

![ISP router IPv4 port mapping for WireGuard UDP 32541 to OpenWrt WAN IP.](/images/openwrt/openwrt-isp-router-port-mapping-wireguard.png)

### Faut-il mettre OpenWrt en DMZ ?

En général, non. Une redirection ciblée WireGuard est plus propre.

## Etape 5 : gérer l'IP publique avec DDNS

Même avec une IP stable, je préfère un nom DNS fixe.

Dans mon cas, j'utilise mon propre domaine avec DNS sur Cloudflare (DynHost indisponible dans mon contexte registrar/manager).

Exemple de FQDN :

- `myprivatevpnXYZ.mmb.sh`

### 5.1 Créer l'enregistrement DNS WireGuard

Dans Cloudflare DNS :

- créez un `A` record pour `myprivatevpnXYZ.mmb.sh`
- mettez une IPv4 temporaire
- `Proxy status = DNS only` (obligatoire pour WireGuard UDP)

### 5.2 Créer un API Token Cloudflare restreint

Token avec droits minimum :

- `Zone - DNS - Edit`
- `Zone - Zone - Read`
- scope limité à la zone `mmb.sh`

### 5.3 Installer DDNS sur OpenWrt

```bash
opkg update
opkg install ddns-scripts luci-app-ddns ddns-scripts-cloudflare ca-bundle ca-certificates
```

Redémarrez OpenWrt après installation du paquet LuCI.

### 5.4 Configurer DDNS dans LuCI

Dans **Services > Dynamic DNS**, créez un service IPv4 :

- `Service Name`: `CloudflareDynDNS`
- `DDNS Service provider`: `cloudflare.com-v4`

![OpenWrt Dynamic DNS page: first step to add a Cloudflare DDNS service.](/images/openwrt/openwrt-ddns-add-service.png)

Dans la popup du service :

- `Enable`: coché
- `Lookup Hostname`: `myprivatevpnXYZ.mmb.sh`
- `Domain`: `myprivatevpnXYZ@mmb.sh` (format `record@zone`)
- `Username`: `Bearer`
- `Password`: `<Cloudflare API Token>`
- `Use HTTP Secure`: coché
- `Path to CA-Certificate`: `/etc/ssl/certs`

![OpenWrt DDNS service popup - Basic Settings for Cloudflare.](/images/openwrt/openwrt-ddns-add-service-basic-settings.png)

Advanced settings :

- si WAN public direct : `IP source = Network`, `Network = wan`
- si OpenWrt derrière box (WAN privée `192.168.x.x`) : `IP source = URL/Web`
- URL fiable : `http://checkip.amazonaws.com` ou `https://api.ipify.org`
- `DNS-Server` vide par défaut
- si les logs montrent une résolution privée (`192.168.x.x`) pour `Lookup Hostname`, forcez un DNS public (`1.1.1.1`)

![OpenWrt DDNS service popup - Advanced Settings for Cloudflare.](/images/openwrt/openwrt-ddns-add-service-advanced-settings.png)

Timer settings :

- `Check Interval = 5 minutes`
- `Force Interval = 72 hours`
- `Error Retry Interval = 60 seconds`
- `Error Max Retry Counter = 0`

![OpenWrt DDNS service popup - Timer Settings for Cloudflare.](/images/openwrt/openwrt-ddns-add-service-timer-settings.png)

Sauvegardez puis démarrez le service.

Note de compatibilité : selon version `ddns-scripts-cloudflare`, certains builds utilisent encore le mode legacy (`X-Auth-Email` / `X-Auth-Key`). Dans ce cas :

- `Username = email Cloudflare`
- `Password = Global API Key`
- `Domain = mmb.sh`
- `Lookup Hostname = myprivatevpnXYZ.mmb.sh`

### 5.5 Valider

Dans LuCI, onglet logs + `Reload`, et en CLI :

```bash
nslookup myprivatevpnXYZ.mmb.sh 1.1.1.1
logread -e ddns
```

## Etape 6 : créer les peers OpenWrt

Dans `wg0 > Peers`, créez un peer par appareil.

Règles :

- une paire de clés par appareil
- une IP tunnel par appareil
- jamais de profil partagé

### Peer Android

Exemple :

- nom : `android-pixel`
- `Allowed IPs` côté serveur : `10.203.0.2/32`

![OpenWrt WireGuard peer settings example for an Android client.](/images/openwrt/openwrt-wg0-android-peer-settings.png)

## Etape 7 : configurer le client Android

Application officielle WireGuard :

- [WireGuard install page (official)](https://www.wireguard.com/install/)

Depuis la page peer OpenWrt :

1. remplissez le peer
2. sauvegardez
3. cliquez sur **Generate configuration...**
4. scannez le QR code depuis l'app Android

![OpenWrt generated Android WireGuard configuration with QR code.](/images/openwrt/openwrt-wg0-android-qr-export.png)

Le profil généré doit inclure au minimum :

- `Address = 10.203.0.2/32`
- `Endpoint = <votre-ddns>:<votre-port-wireguard>`
- `AllowedIPs = 0.0.0.0/0, ::/0` pour full tunnel
- optionnel : `PersistentKeepalive = 25`

Testez en 4G/5G (pas sur le Wi-Fi de la maison) et vérifiez que l'IP publique vue côté client est celle de votre domicile.

### Dépannage Android (retour réel)

Si le tunnel ne monte pas :

1. vérifiez endpoint + port UDP dans le profil généré
2. vérifiez port d'écoute OpenWrt + règle firewall
3. vérifiez la correspondance des clés (public key client côté OpenWrt)
4. sauvegardez le peer avant de générer le QR
5. si besoin, régénérez le peer et rescanez un QR neuf

C'est précisément les points 4 et 5 qui ont résolu mon cas.

## Etape 8 : autres appareils (même principe)

Pour Linux, Windows, macOS ou iOS, la logique est quasiment la même :

1. un peer dédié par appareil
2. export/copie du profil client
3. endpoint/port et clés cohérents
4. `AllowedIPs` selon besoin (LAN only vs full tunnel)
5. validation handshake + trafic

L'UI change selon plateforme, pas la logique WireGuard.

## Tests à faire avant de valider le setup

Je déconseille de s'arrêter au simple fait que “le tunnel se connecte”.

Je vérifierais au minimum :

1. que le handshake WireGuard apparaît sur le routeur
2. que le client atteint bien `10.203.0.1`
3. qu'il atteint le LAN, par exemple `192.168.10.1` puis un hôte interne
4. que le trafic Internet sort bien via la maison
5. que la résolution DNS est cohérente
6. que la reconnexion fonctionne après un changement de réseau mobile / Wi-Fi

Exemples de tests utiles :

```bash
ping 10.203.0.1
ping 192.168.10.1
curl ifconfig.me
```

Si `curl ifconfig.me` renvoie l'IP publique de votre domicile lorsque le tunnel est actif, le routage Internet est probablement correct.

## Problèmes courants et dépannage

### Pas de handshake

Vérifiez :

- endpoint public et port UDP
- redirection de port sur la box FAI
- règle firewall OpenWrt d'entrée UDP
- éventuel CGNAT côté FAI
- clé publique du routeur utilisée côté client

### Handshake OK mais LAN inaccessible

Vérifiez :

- forwarding `vpn -> lan`
- `Allowed IPs` du peer côté OpenWrt
- sous-réseau LAN déclaré côté client
- absence de collision d'adressage

### LAN OK mais pas d'Internet via tunnel

Vérifiez :

- `AllowedIPs = 0.0.0.0/0` côté client
- forwarding `vpn -> wan`
- NAT/masquerade sur zone `wan`
- DNS client cohérent

### DNS incohérent ou fuite DNS

Vérifiez :

- valeur `DNS` dans le profil client
- accessibilité du DNS via tunnel
- éventuel override DNS par Android

### Fonctionne en Wi-Fi mais pas en data mobile

Testez :

- `PersistentKeepalive = 25`
- un autre réseau externe
- éventuel filtrage UDP côté opérateur mobile

## Note IPv6

`::/0` dans `AllowedIPs` est pertinent uniquement si vous voulez réellement faire passer IPv6 dans le tunnel.

Si votre setup IPv6 n'est pas complet, mieux vaut retirer `::/0` plutôt que laisser un comportement partiel ambigu.

## Full tunnel : utile, mais pas gratuit

Avantages :

- accès à des ressources filtrées sur IP maison
- cohérence d'usage en déplacement
- meilleure sécurité sur réseaux non fiables

Limites :

- dépendance à la connexion maison
- upload domestique limitant
- panne électrique maison = plus de sortie Internet
- intérêt variable selon usages

## Bonnes pratiques sécurité

1. une clé par appareil, révocation immédiate en cas de perte
2. sous-réseau VPN distinct du LAN
3. règles firewall minimales nécessaires
4. désactiver UPnP si inutile
5. maintenir OpenWrt à jour
6. sauvegardes exportées de la config
7. limiter l'exposition des interfaces d'administration
8. décider explicitement de la stratégie IPv6

## Références

- [WireGuard Quick Start](https://www.wireguard.com/quickstart/)
- [OpenWrt documentation](https://openwrt.org/docs/start)
- [OpenWrt WireGuard package page](https://openwrt.org/packages/pkgdata/luci-proto-wireguard)
- [OpenWrt Dynamic DNS client](https://openwrt.org/docs/guide-user/services/ddns/client)
- [Cloudflare API tokens](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)

## Pourquoi pas Tailscale ou ZeroTier ?

Ce guide cible un WireGuard auto-géré sur OpenWrt. Tailscale/ZeroTier peuvent simplifier la mise en route (NAT traversal, enrôlement), mais impliquent une couche de contrôle externe.

Je préfère OpenWrt + WireGuard quand je veux :

- contrôle direct de la config routeur
- comportement réseau prévisible
- full tunnel maîtrisé de bout en bout
- architecture proche du WireGuard standard

## Conclusion

WireGuard sur OpenWrt derrière une box FAI est totalement faisable dans un contexte domestique. Le plus important n'est pas l'installation du protocole en elle-même, mais la cohérence globale :

- adressage propre
- firewall clair
- redirections de ports correctes
- gestion réaliste de l'IP publique
- distinction nette entre accès LAN et full tunnel

Pour mon besoin, c'est plus propre que d'exposer des services internes un par un, et plus flexible que des ouvertures de ports ad hoc.
