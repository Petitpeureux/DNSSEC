# Configuration d'un Serveur Web Sécurisé avec Nginx

- Dans le paysage numérique actuel, la **sécurité des sites web** est devenue une exigence fondamentale. Ce projet vous guide à travers la configuration complète d'un serveur web sécurisé utilisant **Nginx**, le deuxième serveur web le plus utilisé au monde (après Apache).

#### Objectifs Principaux

- **Mise en place d'une connexion HTTPS** avec redirection automatique depuis HTTP
- **Renforcement de la sécurité** via :

  - Certificats SSL/TLS
  - Headers de protection
  - Pare-feu UFW

- **Structure propre** et organisée pour une maintenance facile
- **Monitoring** via des fichiers de logs dédiés

#### Schéma d'Architecture Global

![nginx](/assets/nginx.png)

- Ce projet vous fournira une **base solide** pour déployer des sites web professionnels tout en maîtrisant les aspects critiques de la sécurité web moderne. Chaque étape est expliquée en détail avec des commandes précises et des schémas visuels

- Mise à jour des listes de paquets

```sh
sudo apt update
```

- Installation de Nginx, OpenSSL et UFW

```sh
sudo apt install -y nginx openssl ufw
```

- Architecture de base

```sh
[Votre Machine]
├── Nginx (Serveur Web)
├── OpenSSL (Cryptographie)
└── UFW (Pare-feu)
```

- Création de la structure du site
  - Création des répertoires

```sh
sudo mkdir -p /var/www/mon_site
sudo mkdir -p /etc/nginx/ssl
```

- Création d'une page HTML de test

```sh
sudo nano /var/www/mon_site/index.html
```

```sh
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Mon Site</title>
</head>
<body>
    <h1>Bienvenue sur un site sécurisé</h1>
</body>
</html>
```

- Structure des fichiers

```sh
/var/www/mon_site/
└── index.html

/etc/nginx/
└── ssl/
    ├── mon_site.crt
    └── mon_site.key
```

- Génération des certificats SSL

```sh
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/nginx/ssl/mon_site.key \
-out /etc/nginx/ssl/mon_site.crt \
-subj "/C=FR/ST=IDF/L=Paris/O=Entreprise/CN=YOUR_IP_ADDRESS"
```

- Explication des options

  - `-x509`: Crée un certificat auto-signé
  - `-nodes`: Pas de phrase de passe pour la clé
  - `-days 365`: Validité d'un an
  - `-newkey rsa:2048`: Clé RSA de 2048 bits

- Configuration de Nginx
  - Création du fichier de configuration

```sh
sudo nano /etc/nginx/sites-available/mon_site
```

- Contenu du fichier de configuration

```sh
server {
    listen 80;
    server_name YOUR_IP_ADDRESS;

    # Redirection vers HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name YOUR_IP_ADDRESS;

    # Racine du site
    root /var/www/mon_site;
    index index.html;

    # Certificats SSL
    ssl_certificate /etc/nginx/ssl/mon_site.crt;
    ssl_certificate_key /etc/nginx/ssl/mon_site.key;

    # Headers de sécurité
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy "no-referrer-when-downgrade";
    add_header Content-Security-Policy "default-src 'self';";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        try_files $uri $uri/ =404;
    }

    # Logs
    access_log /var/log/nginx/mon_site.access.log;
    error_log /var/log/nginx/mon_site.error.log;
}
```

- Flux des requêtes

```sh
Requête HTTP ──▶ Redirection HTTPS ──▶ Serveur sécurisé
                   (Port 80)              (Port 443)
```

- Activation du site

- Création du lien symbolique

```sh
sudo ln -s /etc/nginx/sites-available/mon_site /etc/nginx/sites-enabled/
```

- Suppression de la configuration par défaut

```sh
sudo rm /etc/nginx/sites-enabled/default
```

- Test de la configuration

```sh
sudo nginx -t
```

- Rechargement de Nginx

```sh
sudo systemctl reload nginx
```

- Sécurisation avec UFW (Uncomplicated Firewall)

- Autorisation des connexions SSH

```sh
sudo ufw allow 'OpenSSH'
```

- Ouverture des ports HTTP et HTTPS

```sh
sudo ufw allow 80
sudo ufw allow 443
```

- Activation du pare-feu

```sh
sudo ufw enable
```

- Table des règles UFW

```sh
Port    Protocole    Action
22      TCP          ALLOW (OpenSSH)
80      TCP          ALLOW (HTTP)
443     TCP          ALLOW (HTTPS)
```

- Vérification finale
  - Test avec curl (option -k pour ignorer le certificat auto-signé)

```sh
curl -k https://YOUR_IP_ADDRESS
```

#### Schéma récapitulatif

```sh
[Internet]
    │
    ▼
[UFW Pare-feu] ← Ports 80/443 ouverts
    │
    ▼
[Nginx]
├─ Port 80: Redirection HTTPS
└─ Port 443: Site sécurisé
    │
    ▼
[/var/www/mon_site]
```

#### Points Clés à Retenir

- **HTTPS obligatoire**: Toutes les requêtes HTTP sont redirigées vers HTTPS
- **Certificats SSL**: Même auto-signés, ils permettent le chiffrement
- **Headers de sécurité**: Protection contre diverses attaques web
- **Pare-feu**: Seuls les ports nécessaires sont ouverts