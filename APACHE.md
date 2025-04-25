# Serveur Web Apache avec HTTPS sur Ubuntu

- **Apache HTTP Server** est le serveur web open-source le plus répandu (40% des sites mondiaux en 2024).

- Dans ce guide, nous allons configurer un `serveur web Apache` sécurisé avec `HTTPS` :

  - 1 - Installation d'Apache
  - 2 - Configuration d'un Virtual Host
  - 3 - Mise en place de SSL/TLS
  - 4 - Redirection HTTP→HTTPS

#### Schéma d'architecture

```sh
[Client Internet]
       │
       ├──(HTTP)──▶ [Apache:80] ────┐
       │                            │ (Redirection)
       └──(HTTPS)─▶ [Apache:443] ◀──┘
                          │
                          ▼
           [/var/www/cfitech.it.local]
                 (Contenu web)
```

#### Fonctionnement clé :

- **Couche Réseau** : Apache écoute sur les ports 80 (HTTP) et 443 (HTTPS)
- **Moteur SSL** : Chiffrement via OpenSSL
- **Gestion des requêtes** :

  - Analyse des en-têtes HTTP
  - Routage vers le bon Virtual Host

#### Installation et Configuration d'Apache

- Mise à jour du Système

```sh
sudo apt-get update
```

- Installation d’Apache

```sh
sudo apt install apache2 -y
```

- Vérification

```sh
sudo systemctl status apache2
```

- Sortie attendue

```sh
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: active (running) since ...
```

- Si Apache ne démarre pas

```sh
sudo systemctl start apache2
```

- Configuration du Site Web
  - Création du Répertoire du Site

```sh
sudo mkdir -p /var/www/cfitech.it.local
```

- `/var/www/` : Dossier par défaut des sites web Apache.
- `-p `: Crée les sous-dossiers nécessaires.

- Modification des Permissions

```sh
sudo chown -R $USER:$USER /var/www/cfitech.it.local
sudo chmod -R 755 /var/www/cfitech.it.local
```

- `chown -R $USER:$USER` : Change le propriétaire vers l’utilisateur actuel.
- `chmod -R 755` : Donne les droits :

- Création d’une Page d’Accueil

```sh
sudo nano /var/www/cfitech.it.local/index.html
```

- Contenu

```sh
<h1>Si tu en as vraiment la volonté, tu peux y arriver</h1>
```

- Configuration du Virtual Host
  - Création du Fichier de Configuration

```sh
sudo nano /etc/apache2/sites-available/cfitech.it.local.conf
```

- Contenu

```sh
<VirtualHost *:80>
    ServerAdmin cfitech@cfitech.local
    ServerName cfitech.it.local
    ServerAlias www.cfitech.it.local
    DocumentRoot /var/www/cfitech.it.local
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- `<VirtualHost *:80>` : Écoute sur le port HTTP (80).
- `ServerName` : Nom de domaine principal.
- `ServerAlias` : Autres noms de domaine associés.
- `DocumentRoot` : Emplacement des fichiers du site.
- `ErrorLog` & `CustomLog` : Fichiers de logs.

- Activation du Site

```sh
sudo apache2ctl configtest  # Vérifie la syntaxe
sudo systemctl restart apache2
```

##### Configuration HTTPS (SSL/TLS)

- Activation du Module SSL

```sh
sudo a2enmod ssl
sudo systemctl restart apache2
```

- Génération d’un Certificat Auto-Signé

```sh
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/apache-selfsigned.key \
    -out /etc/ssl/certs/apache-selfsigned.crt
```

- Paramètres à entrer

- **Country Name** : `BE` (Belgique)
- **State or Province** : BXL (Bruxelles)
- **Locality** : `Koekelberg`
- **Organization** : `Cfitech`
- **Organizational Unit** : `IT`
- **Common Name** : `Sys-Admin`
- **Email** : `YOUR EMAIL`

- Configuration du Virtual Host HTTPS
  - Modifier le fichier :

```sh
sudo nano /etc/apache2/sites-available/cfitech.it.local.conf
```

- Nouveau contenu

```sh
<VirtualHost *:80>
    ServerName cfitech.it.local
    Redirect / https://YOUR_IP_ADDRESS/  # Redirige HTTP → HTTPS
</VirtualHost>

<VirtualHost *:443>
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

    ServerAdmin cfitech@cfitech.local
    ServerName cfitech.it.local
    ServerAlias www.cfitech.it.local
    DocumentRoot /var/www/cfitech.it.local
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- Finalisation

```sh
sudo apache2ctl configtest  # Vérifie la syntaxe
sudo systemctl reload apache2
```

- Test d’Accès

- **HTTP** : `http://YOUR_IP_ADDRESS/` → Doit rediriger vers HTTPS.
- **HTTPS** : `https://YOUR_IP_ADDRESS/` → Affiche la page avec un avertissement (certificat auto-signé).

#### Schéma Réseau Final

```sh
[Client]
   │
   ├─(HTTP)─> [Apache:80] → Redirige vers HTTPS
   │
   └─(HTTPS)─> [Apache:443] → Site sécurisé (SSL)
                  │
                  └─> /var/www/cfitech.it.local
```

- `Remarque`: Le serveur est maintenant opérationnel avec HTTPS forcé !