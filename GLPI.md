# GLPI (Gestionnaire Libre de Parc Informatique)

- GLPI est un outil open-source de gestion des actifs informatiques (IT Service Management) qui permet de gérer les tickets, les inventaires matériels/logiciels, et bien plus encore.

- Gestion des Actifs Informatiques (ITAM)

  - Inventaire automatique (PC, serveurs, imprimantes) via FusionInventory
  - Gestion des licences logicielles (alertes d'expiration, conformité)
  - CMDB (Base de données de configuration) pour tracer les dépendances entre équipements

- Gestion des Tickets (Helpdesk)

  - Création de tickets `incidents/demandes` avec catégorisation
  - Attribution aux techniciens selon les compétences
  - Tableaux de bord pour suivre les temps de résolution

- Gestion des Projets et Tâches

  - Planification d’interventions `gestion des délais/priorités`
  - Suivi des tâches avec notifications

- Automatisation (via plugins)

  - Intégration avec FusionInventory (découverte automatique des équipements)
  - Intégration `LDAP/AD` pour l’authentification centralisée
  - Rapports automatisés (PDF, CSV)
  - Synchronisation avec les outils de monitoring `Nagios, Zabbix`

- Conformité & Sécurité
  - Gestion des licences logicielles
  - Audit des configurations
  - Détection de vulnérabilités (via plugins)

## Pourquoi utiliser GLPI ?

- Gratuit (contrairement à ServiceNow, BMC Remedy, etc.)
- Personnalisable (plugins, thèmes, workflows)
- Communauté active (mises à jour fréquentes)
- Multi-utilisateurs (rôles et permissions avancés)

## Cas Concrets d'Utilisation

- **Helpdesk**
  - `Scénario` : Un employé ouvre un ticket pour "réseau lent" → Le ticket est routé vers l’équipe réseau avec SLA de 2h.
  - `Bénéfice` : Réduction de 40% du temps de résolution (historique des solutions passées).
- **Inventaire**

  - `Scénario` : Scan automatique des nouveaux PC branchés sur le réseau → Alerte si logiciel non autorisé détecté.
  - `Bénéfice` : Inventaire temps réel sans intervention manuelle.

- **Licences**
  - `Scénario` : Alerte automatique 30 jours avant l’expiration des licences Microsoft.
  - `Bénéfice` : Évite les amendes pour non-conformité.

## Scénario quotidien avec GLPI

- **Un employé a un problème avec son imprimante.**
  - Il se connecte au portail GLPI et crée un ticket d’incident.
- **Le technicien reçoit une notification.**
  - Il voit le ticket, l’attribue à lui-même et planifie une intervention.
- **Il consulte l’équipement concerné dans GLPI.**
  - Il voit l’historique : date d’achat, dernier entretien, garantie, etc.
- **Il résout le problème et clôture le ticket.**
  - L’utilisateur est notifié automatiquement.
- **L’intervention est archivée automatiquement.**
  - Cela permet de suivre les performances, les équipements problématiques, etc.
- **L’entreprise analyse les rapports.**
  - Elle remarque que certaines imprimantes tombent souvent en panne → décision d’achat de nouveau matériel.

## Schéma simplifié du fonctionnement de GLPI

```sh
Utilisateurs                       Technicien IT                  Admin / Manager
     │                                  │                                │
     ▼                                  ▼                                ▼
Création de tickets  ─────►  Attribution / Résolution  ─────►  Suivi / Reporting
     │                                  │                                │
     ▼                                  ▼                                ▼
Matériel lié aux tickets  ◄────  Gestion du parc  ◄────  Contrats, coûts, licences
```

## Installation & Configuration

- Avant toute installation, il est essentiel de mettre à jour les paquets

```sh
sudo -i

apt update && apt -y dist-upgrade
```

- Installation et configuration de SSH, pour sécuriser l'accès distant

```sh
apt -y install openssh-server

nano /etc/ssh/sshd_config
```

- Modification dans le fichier

```sh
PermitRootLogin yes
```

- Puis redémarrer le service

```sh
systemctl restart sshd
```

- Installation d’Apache2
  - GLPI nécessite un serveur web. Nous utiliserons Apache

```sh
apt install apache2 -y
```

- Installation de PHP 8.2
  - GLPI requiert PHP. Nous ajoutons le dépôt Ondřej Surý

```sh
sudo apt install software-properties-common -y
```
```sh
sudo add-apt-repository ppa:ondrej/php -y
```
```sh
sudo apt update
```

- Installation des modules PHP 8.2

```sh
sudo apt install -y \
    php8.2-cli \
    php8.2-fpm \
    php8.2-common \
    php8.2-mysql \
    php8.2-curl \
    php8.2-gd \
    php8.2-intl \
    php8.2-mbstring \
    php8.2-xml \
    php8.2-zip \
    php8.2-bcmath \
    php8.2-gmp \
    php8.2-opcache \
    php8.2-apcu
```

- Vérification de PHP

```sh
php8.2 -v
php8.2 -m
```

- Installation de dépendances supplémentaires

```sh
sudo apt update && sudo apt install -y \
    libapache2-mod-php8.2 \
    hunspell \
    certbot \
    imagemagick \
    unzip \
    php8.2-opcache
```

- Vérification et configuration PHP

```sh
php -v
ls /etc/php/
```

- Création d'un fichier PHP de test

```sh
nano /var/www/html/main.php
```

- Contenu

```sh
<?php phpinfo(); ?>
```

- Puis

```sh
systemctl restart apache2
systemctl status apache2
```

- Accès via : `http://<IP_SERVER>/main.php`

- Installation de MariaDB (MySQL)
  - GLPI utilise une base de données

```sh
apt -y install mariadb-server
systemctl restart mariadb
```

- Sécurisation de MariaDB

```sh
mysql_secure_installation
```

- Réponses

```sh
Switch to unix_socket authentication [Y/n] n
Change the root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

- Puis

```sh
systemctl restart mariadb
```

- Création de la base de données GLPI

```sh
mysql -u root
```

- Exécuter dans MySQL

```sh
show databases;
create database glpi;
create user 'admin'@localhost identified by 'cfitech';
grant all privileges on glpi.* to admin@localhost;
flush privileges;
exit
```

- Puis

```sh
systemctl restart mariadb
```

- Téléchargement et extraction de GLPI
  - Depuis : [https://github.com/glpi-project/glpi/releases?page=2](https://github.com/glpi-project/glpi/releases?page=2)

```sh
wget https://github.com/glpi-project/glpi/releases/download/10.0.6/glpi-10.0.6.tgz
tar -xzf glpi-10.0.6.tgz -C /var/www/html/
ls /var/www/html/
```

- Configuration d'Apache pour GLPI

```sh
nano /etc/apache2/sites-available/000-default.conf
```

- Contenu

```sh
<VirtualHost *:80>
    DocumentRoot /var/www/html/glpi
</VirtualHost>
```

- Puis droits d’accès

```sh
chown -R www-data:www-data /var/www/html/glpi
ls -l /var/www/html
```

<!-- - Installation et activation de SELinux

```sh
sudo apt install selinux-basics selinux-policy-default auditd -y

sudo selinux-activate

nano /etc/selinux/config
```

- Modification

```sh
SELINUX=Enforcing
``` -->

- Installation de modules PHP supplémentaires

```sh
sudo apt install php8.2-ldap php8.2-imap php8.2-xmlrpc -y
systemctl restart apache2
```

- Installation de GLPI via l'interface web
  - Accès à : `http://<IP_SERVER>/install/install.php`
  - Sélectionner la langue et suivre l’assistant
  - Entrer les informations de la base de données
  - Valider l’installation

```sh
Serveur : localhost
Utilisateur : admin
Mot de passe : cfitech
Base de données : glpi
```

- Sécurisation de GLPI
  - Supprimer le script d’installation

```sh
rm -fr /var/www/html/glpi/install/install.php
```

- Modifier `php.ini` pour renforcer la sécurité

```sh
nano /etc/php/8.2/apache2/php.ini
```

- Ajouter/modifier

```sh
session.cookie_httponly = on
```

- Redémarrer Apache

```sh
systemctl restart apache2
```

- Accès à GLPI
  - L’interface est disponible à l’adresse `http://<IP_SERVER>`
  - Identifiant par défaut : `glpi`
  - Mot de passe : `glpi`

<!-- ![GLPI](/assets/glpi.png) -->

- Installation du Plugin FusionInventory

```sh
# Se placer dans le dossier des plugins GLPI
cd /var/www/html/glpi/plugins

# Télécharger la dernière version du plugin
wget https://github.com/fusioninventory/fusioninventory-for-glpi/releases/download/glpi10.0.6%2B1.1/fusioninventory-10.0.6+1.1.tar.bz2

# Extraire l'archive
tar -xjf fusioninventory-10.0.6+1.1.tar.bz2

# Configurer les permissions
chown -R www-data:www-data fusioninventory
```

- Configuration Apache

```sh
# Créer la configuration virtuelle
nano /etc/apache2/sites-available/glpi.conf
```

- Ajoutez cette configuration

```sh
<VirtualHost *:80>
    DocumentRoot /var/www/html/glpi
    <Directory /var/www/html/glpi>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

- Puis exécuter

```sh
a2ensite glpi.conf
a2dissite 000-default.conf
systemctl restart apache2
```

```sh
curl -I http://localhost/plugins/fusioninventory/front/plugin_fusioninventory.communication.php
```

- Configurez le cron pour les tâches automatique (optionnel mais recommandé)

```sh
crontab -e
```

- Ajouter cette ligne

```sh
*/1 * * * * php /var/www/html/glpi/front/cron.php
```

```sh
/etc/init.d/cron restart
systemctl restart cron
```

- Activation dans GLPI
  - Se connecter à GLPI (http://<votre-ip>)
  - Aller dans `Configuration > Plugins`
  - Activer `FusionInventory`
  - Aller dans `Configuration > Action automatique`
  - Rechercher `taskscheduler` et cliquer sur `Exécuter`

#### Installation des Agents

- Pour **Windows**
  - Télécharger l'agent depuis [https://github.com/fusioninventory/fusioninventory-agent/releases/tag/2.6](https://github.com/fusioninventory/fusioninventory-agent/releases/tag/2.6)
  - Installer l'agent
  - Configurer `C:\Program Files\FusionInventory-Agent\agent.cfg` avec :

```sh
Mode servers = http://<IP_SERVER>/plugins/fusioninventory/front/plugin_fusioninventory.communication.php
```

- Redémarrer le service

```sh
net stop FusionInventory-Agent && net start FusionInventory-Agent
```

- Pour **Linux (Debian/Ubuntu)**

```sh
wget https://github.com/fusioninventory/fusioninventory-agent/releases/download/2.6/fusioninventory-agent_2.6-1_all.deb
sudo apt --fix-broken install
sudo apt install hwdata


sudo apt install libnet-cups-perl libnet-ip-perl libwww-perl libparse-edid-perl \
libproc-daemon-perl libuniversal-require-perl libfile-which-perl libxml-treepp-perl \
libxml-xpath-perl libyaml-perl libtext-template-perl libhttp-daemon-perl \
libyaml-tiny-perl libsocket-getaddrinfo-perl

sudo dpkg -i fusioninventory-agent_2.6-1_all.deb
sudo apt install -f
```

- Configuration

```sh
sudo nano /etc/fusioninventory/agent.cfg
```

- Ajouter exactement

```sh
<server url="http://<IP_SERVER>/plugins/fusioninventory/front/plugin_fusioninventory.communication.php"/>
```

- Puis

```sh
sudo systemctl enable fusioninventory-agent
sudo systemctl start fusioninventory-agent
```

- Vérification dans GLPI

  - Dans GLPI, aller dans `Plugins > FusionInventory > Inventaire`
  - Les équipements devraient apparaître dans les 5-10 minutes

- Pour gérer le **Marketplace**
  - Aller dans `Configuration > General > GLPI Network`
  - Activer/désactiver selon besoins
