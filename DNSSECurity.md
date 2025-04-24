# DNS Sécurisé avec BIND9 et DNSSEC (Version Simplifiée)

## Objectif
Sécuriser les requêtes DNS grâce à DNSSEC avec une configuration moderne et simplifiée sous BIND9.

---

## 🚀 Avantages de cette configuration

- ✅ Prévention des attaques MITM (Man-In-The-Middle)
- ✅ Configuration adaptée à tous les registrars
- ✅ Automatisation de la validation DNSSEC grâce à `dnssec-validation auto`

---

## 🚧 Préparation du système

### Mise à jour et installation
```sh
sudo apt update && sudo apt upgrade -y
sudo apt install bind9 bind9utils bind9-doc -y
```

### Activation de BIND9
```sh
sudo systemctl start bind9
sudo systemctl enable bind9
sudo systemctl status bind9
```

### Pare-feu
```sh
sudo ufw enable
sudo ufw allow 53
sudo ufw reload
```

### Configuration IP statique
Modifiez `/etc/netplan/01-netcfg.yaml` :
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 172.28.0.2/16
      routes:
        - to: default
          via: 172.28.0.1
      nameservers:
        addresses: [172.28.0.2]
```
Appliquez la configuration :
```sh
sudo netplan apply
```

---

## 🛠️ Configuration de BIND9 (Serveur Primaire)

### `/etc/bind/named.conf.options`
```conf
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    dnssec-validation auto;  // Validation DNSSEC automatique

    listen-on { 172.28.0.2; };
    listen-on-v6 { none; };

    allow-query { 172.28.0.0/16; };
    allow-recursion { 172.28.0.0/16; };
};
```
Redémarrer le service :
```sh
sudo systemctl restart bind9
```

---

## 🔍 Zones DNS

### `/etc/bind/named.conf.local`
```conf
zone "cfitech-it.com" {
    type master;
    file "/etc/bind/db.cfitech-it.com";
    allow-transfer { 172.28.0.3; };
    also-notify { 172.28.0.3; };
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.rev.cfitech-it.com";
};
```

### Zone directe : `/etc/bind/db.cfitech-it.com`
```dns
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025042401 ; Serial
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.
@       IN      A       172.28.0.2

ns      IN      A       172.28.0.2
ns2     IN      A       172.28.0.3
www     IN      CNAME   ns
router  IN      A       172.28.0.1
```

### Zone inversée : `/etc/bind/db.rev.cfitech-it.com`
```dns
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025042401 ; Serial
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.

2       IN      PTR     ns.cfitech-it.com.
3       IN      PTR     ns2.cfitech-it.com.
```

### Vérification
```sh
sudo named-checkconf
sudo named-checkzone cfitech-it.com /etc/bind/db.cfitech-it.com
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.rev.cfitech-it.com
```
Redémarrage :
```sh
sudo systemctl restart bind9
```

---

## 🔒 DNSSEC : validation automatique

La ligne `dnssec-validation auto` dans `named.conf.options` permet à BIND de valider automatiquement les signatures DNSSEC à l'aide des ancres de confiance incluses. Cela supprime le besoin de gérer manuellement des clés.

### Test
```sh
dig +dnssec www.cfitech-it.com @172.28.0.2
dig -x 172.28.0.2 +dnssec @172.28.0.2
```

---

## 📄 Conclusion

Votre serveur BIND9 est maintenant prêt pour la production avec DNSSEC activé, sans complexité inutile. Idéal pour des environnements sécurisés, maintenables et fiables.

