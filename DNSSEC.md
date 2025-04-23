# DNS

- La configuration d'une infrastructure DNS fiable et sécurisée est essentielle pour assurer la disponibilité et la performance des services réseau. Un système DNS bien conçu repose sur une architecture primaire/secondaire (maître/esclave) qui garantit la redondance, la répartition de charge et la continuité de service en cas de défaillance du serveur primaire.

- Dans ce guide, nous allons déployer une solution DNS complète avec :

  - **Un serveur DNS primaire** (maître) qui héberge les zones principales
  - **Un serveur DNS secondaire** (esclave) qui réplique les données pour assurer la haute disponibilité
  - **Des zones directes (A, CNAME) et inverses (PTR)** correctement configurées
  - **Une sécurité renforcée** (transferts de zone restreints, DNSSEC optionnel)
  - **Des mécanismes de vérification** pour valider la configuration

- Cette solution utilise **BIND9** sous **Ubuntu**, la référence pour les serveurs DNS, et suit les meilleures pratiques en matière de sécurité et de performance.

#### 1. Architecture Réseau

- Les clients interrogent indifféremment le primaire ou le secondaire.
- Le secondaire réplique les zones via des transferts AXFR(Full Zone Transfer) / IXFR(Incremental Zone Transfer) sécurisés.
- Les requêtes externes (hors zone) sont forwardées vers Google DNS.

![dns](/assets/dns1.png)

#### 2. Flux des Requêtes

- Priorité au serveur primaire, bascule automatique vers le secondaire si échec.
- Mécanisme de résolution hiérarchique (locale → forwarders → root).

![dns](/assets/dns2.png)

#### 3. Zones DNS

- La zone directe gère la résolution **nom → IP**.
- La zone inverse gère la résolution **IP → nom** (essentielle pour les vérifications anti-spam).

![dns](/assets/dns3.png)

#### Table des matières

- **Redondance** : Deux serveurs minimum (RFC 2182).
- **Sécurité** : Transferts de zone restreints par IP (allow-transfer).
- **Maintenabilité** : Serial incrémenté à chaque modification.
- **Performance** : Cache optimisé (TTL) et forwarders configurés.

#### Serveur Primaire (192.168.1.2)

- Mise à jour du système

```sh
sudo apt update && sudo apt upgrade -y
```

- Installation de Bind9

```sh
sudo apt install bind9 bind9utils bind9-doc -y
```

- Activation du service

```sh
sudo systemctl enable bind9
sudo systemctl start bind9
```

- Configuration du pare-feu

```sh
sudo ufw allow 53
sudo ufw reload
```

- Configuration réseau (IP statique)
  - Configuration du serveur primaire

```sh
sudo nano /etc/netplan/01-netcfg.yaml
```

- Configuration IP statique

```sh
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [192.168.1.2/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.2, 192.168.1.3]
```

- Appliquez la configuration

```sh
sudo netplan apply
```

- Configuration des options Bind9

```sh
sudo nano /etc/bind/named.conf.options
```

- Ajoutez ou modifiez les options suivantes

```sh
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    dnssec-validation auto;

    listen-on { any; };
    listen-on-v6 { any; };
    allow-query { any; };
    allow-transfer { 192.168.1.3; }; // Seulement vers le secondaire
    recursion yes;
};
```

- Redémarrez Bind9 pour appliquer les modifications

```sh
sudo systemctl restart bind9
```

- Configuration des zones

```sh
sudo nano /etc/bind/named.conf.local
```

- Contenu du fichier

```sh
zone "cfitech-it.com" {
    type master;
    file "/etc/bind/db.cfitech-it.com";
    allow-transfer { 192.168.1.3; };  # Autorise le transfert vers le serveur secondaire.
    also-notify { 192.168.1.3; };     # Notifie le serveur secondaire des mises à jour.
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.rev.cfitech-it.com";
    allow-transfer { 192.168.1.3; };
};
```

- Créez un fichier pour la zone directe

```sh
sudo cp /etc/bind/db.local /etc/bind/db.cfitech-it.com
```

```sh
sudo nano /etc/bind/db.cfitech-it.com
```

- Contenu du fichier

```sh
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025040201 ; Serial   (à incrémenter lors des modifications)
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

; Serveurs DNS
@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.

; Enregistrements A
@       IN      A       192.168.1.2  ; Serveur primaire.
ns      IN      A       192.168.1.2
ns2     IN      A       192.168.1.3  ; Serveur secondaire.
www     IN      CNAME   ns           ; Alias vers ns.
router  IN      A       192.168.1.1  ; Routeur local.
```

- Créez un fichier pour la zone inversée

```sh
sudo cp /etc/bind/db.local /etc/bind/db.rev.cfitech-it.com
```

- Zone inverse

```sh
sudo nano /etc/bind/db.rev.cfitech-it.com
```

```sh
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025040201 ; Serial
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.

2       IN      PTR     ns.cfitech-it.com.
3       IN      PTR     ns2.cfitech-it.com.
```

- Avant de redémarrer Bind9, vérifiez les fichiers de configuration

- Vérifie la syntaxe générale.

```sh
sudo named-checkconf
```

- Vérifie la zone directe.

```sh
sudo named-checkzone cfitech-it.com /etc/bind/db.cfitech-it.com
```

- Vérifie la zone inversée.

```sh
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.rev.cfitech-it.com
```

- Redémarrez Bind9 après vérification

```sh
sudo systemctl restart bind9 && sudo systemctl status bind9
```

#### Configuration du Serveur Secondaire (192.168.1.3)

- Mise à jour du système

```sh
sudo apt update && sudo apt upgrade -y
```

- Installation de Bind9

```sh
sudo apt install bind9 bind9utils bind9-doc -y
```

- Activation du service

```sh
sudo systemctl enable bind9
sudo systemctl start bind9
```

- Configuration du pare-feu

```sh
sudo ufw allow 53
sudo ufw reload
```

- Configuration IP statique

```sh
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [192.168.1.3/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.2, 192.168.1.3]
```

```sh
sudo netplan apply
```

- Configuration des options Bind9

```sh
sudo nano /etc/bind/named.conf.options
```

```sh
options {
    directory "/var/cache/bind";
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { any; };
    allow-query { any; };
    recursion yes;
};
```

- Configuration des zones esclaves

```sh
sudo nano /etc/bind/named.conf.local
```

```sh
zone "cfitech-it.com" {
    type slave;
    file "/var/cache/bind/db.cfitech-it.com";
    masters { 192.168.1.2; };  # Adresse IP du serveur primaire.
};

zone "1.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.rev.cfitech-it.com";
    masters { 192.168.1.2; };  # Adresse IP du serveur primaire.
};
```

- Vérification

```sh
sudo named-checkconf
```

- Redémarrage

```sh
sudo systemctl restart bind9
sudo systemctl status bind9
```

- Vérification du transfert de zone

```sh
sudo ls -l /var/cache/bind/
```

- Tests depuis un client

```sh
# Test de résolution directe
nslookup ns.cfitech-it.com 192.168.1.2
nslookup www.cfitech-it.com 192.168.1.3

# Test de résolution inverse
nslookup 192.168.1.2 192.168.1.2
nslookup 192.168.1.3 192.168.1.3

# Test de transfert de zone
dig @192.168.1.2 cfitech-it.com AXFR
dig @192.168.1.3 cfitech-it.com AXFR
```