# Installation et paramétrage d'un serveur web sécurisé dans une VM Debian 11 avec AWS
## Création d'une machine Debian 11 et connection 
Étape 1 : Pour commencer, nous allons créer une instance (une VM) Debian 11 :  
![Création d'une machine debian 11 partie 1](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/cr%C3%A9er_machine_partie_1.png)
Étape 2 : Sélectionnez l'OS Debian, et t2.micro en type d'instance (pour sa gratuité) :
![Création d'une machine debian 11 partie 2](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/cr%C3%A9er_machine_partie_2.png)
Étape 3 : Créez une paire de clés ".pem" pour pouvoir se connecter en ssh, et créez un groupe de sécurité (c'est un firewall) en ouvrant les ports 22, 80 et 443 (pour respectivement ssh, http et https). Une fois tout cela fait, vous pouvez lancer l'instance :  
![Création d'une machine debian 11 partie 3](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/cr%C3%A9er_machine_partie_3.png)
Étape 4 : Une fois l'instance lancée, sélectionnez-la en cliquant sur "Instances" puis l'ID de votre instance :  
![Création d'une machine debian 11 partie 4](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/cr%C3%A9er_machine_partie_4.png)
Étape 5 : Pour des instructions quant à la connexion, sélectionnez "Se connecter" :  
![Création d'une machine debian 11 partie 5](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/cr%C3%A9er_machine_partie_5.png)
Étape 6 : Ouvrez une invite de commande ou un terminal, et positionnez vous dans le répertoire qui contient votre clé ".pem" téléchargé à l'étape 3. Copiez la commande donnée dans l'exemple de la connexion via "Client SSH", et collez-la dans le terminal :  
![Création d'une machine debian 11 partie 6](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/cr%C3%A9er_machine_partie_6.png)
Vous êtes connecté à votre machine Debian via SSH !  
Vous pouvez également accéder au firewall d'AWS pour y ajouter des règles entrantes :  
![Comment accéder au firewall d'AWS](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/configurer_aws_firewall.png)  
Vous pouvez voir toutes les règles à ajouter si vous suivez tout ce tuto, libre à vous de les ajouter maintenant ou au moment où elles vous seront utiles. **Attention, la règle pour le ssh reste sur le port 22. Pour modifier le port qu'utilise le protocole ssh, référez-vous à la partie "[Changer le port SSH](###Changer-le-port-SSH)"** :  
![Comment accéder au firewall d'AWS 2](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/configurer_aws_firewall_partie_2.png)
## Installation et configuration d'un LAMP dans la machine + GLPI et FTP
Premièrement, passez en super utilisateur pour avoir les pleins droits sur la machine. Dans les machines Debian d'AWS, vous n'avez pas le mot de passe pour ce compte, mais le package "sudo" est installé. Vous pouvez donc faire :  
```
sudo su -
```
Faites les mise à jour :  
```
apt update
apt upgrade
```
Installez les paquets nécessaires à la création du LAMP (serveur web : apache, base de données : mariadb, language : php) :  
```
apt install apache2 php libapache2-mod-php mariadb-server php-mysqli php-mbstring php-curl php-gd php-simplexml php-intl php-ldap php-apcu php-xmlrpc php-cas php-zip php-bz2 php-ldap php-imap -y
``` 
Faites une installation sécurisée de mysql (mariadb) :  
```
mysql_secure_installation
```
Ajoutez un mot de passe pour le compte root, puis répondez oui pour toute les questions sauf le changement du mot de passe root (vous venez de le mettre !).  
Créez ensuite une base de données, donnez les droits de cette base à un utilisateur et recharger les privilèges pour que les modifications aient lieu maintenant :  
```
mysql -u root  
CREATE DATABASE nom_base_de_données ;
GRANT ALL PRIVILEGES ON glpidb.* TO 'nom_utilisateur'@'localhost' IDENTIFIED BY 'mot_de_passe';
FLUSH PRIVILEGES;
exit
```
Ensuite, vous allez créer et configurer votre propre fichier de configuration apache 2 :
```
cd /etc/apache2/sites-available
cat 000-default.conf > nom_config.conf
cat default-ssl.conf >> nom_config.conf
```
Puis, avec l'éditeur de votre choix, ajouter ces lignes en dessous de "DocumentRoot /var/www/html" :  
```
<Directory /var/www/html>
   Options Indexes FollowSymLinks
   AllowOverride All
   Require all granted
</Directory>
```
Vous pouvez également changer le chemin pour votre page html pour plus de sécurité. Dans ce cas, mettez le chemin absolu vers son emplacement après les deux "DocumentRoot" et "Directory". Dans mon cas, j'ai créé un utilisateur "debianuser" avec la commande :  
```
adduser debianuser
```
J'ai ensuite créé un dossier "html" dans le dossier de l'utilisateur "debianuser" (je continuerai à utiliser cet exemple pour la suite) :  
```
DocumentRoot /home/debianuser/html
<Directory /home/debianuser/html>
   Options Indexes FollowSymLinks
   AllowOverride All
   Require all granted
</Directory>
```
Activez votre nouvelle configuration d'apache, désactivez les autres, activez l'HTTPS puis redémarrez le service : 
```
a2ensite nom_config.conf
a2dissite 000-default.conf
a2dissite default-ssl.conf
a2enmod ssl
systemctl restart apache2
```
Il est temps de télécharger et installer glpi :  
```
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/10.0.7/glpi-10.0.7.tgz
tar -xvzf glpi-10.0.7.tgz
cp -r glpi/* /home/debianuser/html
```
N'oubliez pas de donner les droits sur vos fichiers et dossiers html à apache :  
```
chown -R www-data /home/debianuser/html
```
Puis redémarrez le service apache une nouvelle fois :  
```
systemctl restart apache2
```
Connectez-vous à votre interface web avec http://votre_ip, et configurer votre glpi comme bon vous semble.  
À présent, installez le service ftp :  
```
apt install vsftpd
```
Et configurez son fichier de configuration. Voici un exemple de fichier :  
```
#
# The default compiled in settings are fairly paranoid. This sample file
# loosens things up a bit, to make the ftp daemon more usable.
# Please see vsftpd.conf.5 for all compiled in defaults.
#
# READ THIS: This example file is NOT an exhaustive list of vsftpd options.
# Please read the vsftpd.conf.5 manual page to get a full idea of vsftpd's
# capabilities.
#
#
# Run standalone?  vsftpd can run either from an inetd or as a standalone
# daemon started from an initscript.
listen=NO
#
# This directive enables listening on IPv6 sockets. By default, listening
# on the IPv6 "any" address (::) will accept connections from both IPv6
# and IPv4 clients. It is not necessary to listen on *both* IPv4 and IPv6
# sockets. If you want that (perhaps because you want to listen on specific
# addresses) then you must run two copies of vsftpd with two configuration
# files.
listen_ipv6=YES
#
# Allow anonymous FTP? (Disabled by default).
#anonymous_enable=YES
#anon_root=/var/ftp/pub
#
# Uncomment this to allow local users to log in.
local_enable=YES
#
# Uncomment this to enable any form of FTP write command.
write_enable=YES
#
# Default umask for local users is 077. You may wish to change this to 022,
# if your users expect that (022 is used by most other ftpd's)
local_umask=022
#
# Uncomment this to allow the anonymous FTP user to upload files. This only
# has an effect if the above global write enable is activated. Also, you will
# obviously need to create a directory writable by the FTP user.
#anon_upload_enable=YES
#
# Uncomment this if you want the anonymous FTP user to be able to create
# new directories.
#anon_mkdir_write_enable=YES
#
# Activate directory messages - messages given to remote users when they
# go into a certain directory.
dirmessage_enable=YES
#
# If enabled, vsftpd will display directory listings with the time
# in  your  local  time  zone.  The default is to display GMT. The
# times returned by the MDTM FTP command are also affected by this
# option.
use_localtime=YES
#
# Activate logging of uploads/downloads.
xferlog_enable=YES
#
# Make sure PORT transfer connections originate from port 20 (ftp-data).
connect_from_port_20=YES
#
# If you want, you can arrange for uploaded anonymous files to be owned by
# a different user. Note! Using "root" for uploaded files is not
# recommended!
#chown_uploads=YES
#chown_username=whoever
#
# You may override where the log file goes if you like. The default is shown
# below.
#xferlog_file=/var/log/vsftpd.log
#
# If you want, you can have your log file in standard ftpd xferlog format.
# Note that the default log file location is /var/log/xferlog in this case.
#xferlog_std_format=YES
#
# You may change the default value for timing out an idle session.
#idle_session_timeout=600
#
# You may change the default value for timing out a data connection.
#data_connection_timeout=120
#
# It is recommended that you define on your system a unique user which the
# ftp server can use as a totally isolated and unprivileged user.
#nopriv_user=ftpsecure
#
# Enable this and the server will recognise asynchronous ABOR requests. Not
# recommended for security (the code is non-trivial). Not enabling it,
# however, may confuse older FTP clients.
#async_abor_enable=YES
#
# By default the server will pretend to allow ASCII mode but in fact ignore
# the request. Turn on the below options to have the server actually do ASCII
# mangling on files when in ASCII mode.
# Beware that on some FTP servers, ASCII support allows a denial of service
# attack (DoS) via the command "SIZE /big/file" in ASCII mode. vsftpd
# predicted this attack and has always been safe, reporting the size of the
# raw file.
# ASCII mangling is a horrible feature of the protocol.
#ascii_upload_enable=YES
#ascii_download_enable=YES
#
# You may fully customise the login banner string:
ftpd_banner=David Banner
#
# You may specify a file of disallowed anonymous e-mail addresses. Apparently
# useful for combatting certain DoS attacks.
#deny_email_enable=YES
# (default follows)
#banned_email_file=/etc/vsftpd.banned_emails
#
# You may restrict local users to their home directories.  See the FAQ for
# the possible risks in this before using chroot_local_user or
# chroot_list_enable below.
chroot_local_user=YES
#
# You may specify an explicit list of local users to chroot() to their home
# directory. If chroot_local_user is YES, then this list becomes a list of
# users to NOT chroot().
# (Warning! chroot'ing can be very dangerous. If using chroot, make sure that
# the user does not have write access to the top level directory within the
# chroot)
allow_writeable_chroot=YES
#chroot_list_enable=YES
# (default follows)
#chroot_list_file=/etc/vsftpd.chroot_list
#
# You may activate the "-R" option to the builtin ls. This is disabled by
# default to avoid remote users being able to cause excessive I/O on large
# sites. However, some broken FTP clients such as "ncftp" and "mirror" assume
# the presence of the "-R" option, so there is a strong case for enabling it.
#ls_recurse_enable=YES
#
# Customization
#
# Some of vsftpd's settings don't fit the filesystem layout by
# default.
#
# This option should be the name of a directory which is empty.  Also, the
# directory should not be writable by the ftp user. This directory is used
# as a secure chroot() jail at times vsftpd does not require filesystem
# access.
secure_chroot_dir=/var/run/vsftpd/empty
#
# This string is the name of the PAM service vsftpd will use.
pam_service_name=vsftpd
#
# This option specifies the location of the RSA certificate to use for SSL
# encrypted connections.
rsa_cert_file=/etc/apache2/apache.pem
rsa_private_key_file=/etc/apache2/apache.pem
ssl_enable=YES
pasv_enable=yes
pasv_min_port=65000
pasv_max_port=65500
```
Vous pouvez ajouter les ports 20 et 21 pour le mode FTP actif, et 65000 à 65500 pour le mode FTP passif.  

## Sécuriser le serveur
### Changer le port SSH
Éditez le fichier de configuration du service ssh, qui a pour chemin /etc/ssh/sshd_config, et décommentez la ligne "Port 22" en modifiant 22 par le port de votre choix :  
![sshd_config](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/sshd_config.png)
N'oubliez pas d'ouvrir le port correspondant dans AWS.
### HTTPS via Certbot
Pour utiliser Certbot, vous devrez avoir un nom de domaine. Pour cela, vous pouvez passer par exemple par [No-IP](https://www.noip.com/). Créez un compte, puis créez votre nom de domaine gratuit en le lien à l'IP publique de votre machine AWS.  
Vous pouvez ensuite suivre les directives du site officiel de [Certbot](https://certbot.eff.org/), sinon, voici les commandes à entrer :  
```
apt install snapd
snap install core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
certbot certonly --apache
```
S'ensuit l'installation de certbot :  
![Installation de Certbot](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/installation_certbot.png)  
N'oubliez pas d'indiquer votre propre nom de domaine (domain name) au lieu de "rmarchal.cefim.ninja".  
Vous pouvez observer les chemins absolus du certificat et de la clé ssl (en l'occurence sur l'image ci-dessus /etc/letsencrypt/live/rmarchal.cefim.ninja/fullchain.pem), retenez-les.  
Éditez à nouveau votre fichier de configuration de votre serveur apache2, pour y indiquer les chemins du certificat et de la clé ssl, ainsi qu'une redirection des requêtes http vers le https, et également le déplacement de :  
```
<Directory /var/www/html>
   Options Indexes FollowSymLinks
   AllowOverride All
   Require all granted
</Directory>
```
De "VirtualHost *:80" vers "VirtualHost \_default\_:443" :
![Fichier de conf "willy"](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/conf_apache_nommée_willy.png)
Vous pouvez créer une tâche automatique qui renouvellera le certificat Certbot automatiquement (dans l'image, une fois tout les 3 mois):  
```
crontab -e
```
![Crontab renouvellement Certbot](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/crontab_certbot.png)
### FTPS (possible seulement si vous avez fait le [HTTPS](###HTTPS-via-Certbot))
Éditez le fichier de configuration du service ftp, et indiquez-y les chemins absolus de votre certificat et clé ssl :  
![Fichier de conf FTP](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/fichier_de_conf_ftp.png)
Redémarrez votre service FTP :  
```
systemctl restart vsftpd
```
Vous pouvez tester votre connexion par exemple via [Filezilla](https://filezilla-project.org/), en allant dans Édition puis Gestionnaire de sites:  
![Connexion en FTPS via Filezilla](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/connexion_ftp_via_filezilla_2.png)
Sélectionnez bien "Connexion FTP explicite sur TLS", et entrez un utilisateur autre que "root" ou "admin" : en effet, vous n'avez pas les mots de passe de ces derniers ! Pour créer un utilisateur, faites :  
```
adduser nom_utilisateur
```
En cliquant sur connexion, vous devriez avoir accès aux fichiers dans le dossier du nouvel utilisateur :  
![Connexion en FTPS via Filezilla 2](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/connexion_ftp_via_filezilla.png)
### Isolations des utilisateurs de la base de données
Pour gérer vos bases de données plus facilement, téléchargez le packet "phpmyadmin", puis activez le site :  
```
apt install phpmyadmin
ln -s /etc/phpmyadmin/apache.conf /etc/apache2/conf-available/phpmyadmin.conf
a2enconf phpmyadmin.conf
systemctl restart apache2
```
Vous pouvez maintenant vous connecter au site avec https://votre_nom_de_domaine/phpmyadmin :  
![Page d'accueil phpmyadmin](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/création_utilisateur_mysql_partie_0.png)
Connectez-vous avec le compte root, puis allez dans Comptes Utilisateurs, puis dans Ajouter un compte utilisateur :  
![Ajouter utilisateur](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/création_utilisateur_mysql_partie_1.png)
Continuez en remplissant les informations, et cochez "Créer une base portant son nom et donner à cet utilisateur tous les privilèges sur cette base" :  
![Configuration de l'utilisateur](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/création_utilisateur_mysql_partie_2.png)
Déconnectez vous avec la deuxième icône (la porte), puis connectez vous avec le nouvel utilisateur. Il n'a accès qu'à sa base de données (ainsi qu'à information_schema, qui est une base de données utilisée par mysql) :  
![Configuration de l'utilisateur 2](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/création_utilisateur_mysql_partie_4.png)
### Backups...
**... de la base de données :**   
Vous allez créer une tâche automatique à l'aide de crontab qui compressera le dossier contenant les bases de données en le sauvegardant dans un dossier qui aura pour nom "ladatedujour_www_backup-fichier.zip.  
Créez d'abord le script :  
```
cd /root
touch backup.sh
chmod +x backup.sh
```
Éditez le fichier backup que vous venez de créer :  
```
#!/bin/bash
#Script de backup auto de la base et du www pour le site user
#V1.1 par T.Cherrier sep 2022

clear
echo Compression :
 zip -rq /home/user/$(date +%Y%m%d%H%M)_www_backup-fichier.zip  /home/user/www
echo Dump de la base
mysqldump -u root -p'dadfba16' --databases user > /home/user/$(date +%Y%m%d%H%M)_www_backup-base.sql
echo Terminé
```
Voilà ce que ça donne avec mon chemin :  
![backup.sh](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/backup_sh.png)
Ouvrez à présent le fichier qui contient les tâches automatiques :  
```
crontab -e
```
Puis créez la tâche qui éxecutera automatiquement le script (dans l'image, tout les jours à 3h du matin) :  
![Crontab backup](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/crontab_backup.png)
**... de la machine virtuelle :**
Pour créer une snapshot de votre machine, allez sur le site AWS et repérez le volume attaché à votre instance :  
![Snapshot partie 1](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/créer_une_snapshot_partie_1.png)
Retournez dans l'onglet "Volume" (sur la droite), sélectionnez-la puis cliquez sur le menu déroulant "Action" pui enfin sur "Créer un instantané" :  
![Snapshot partie 2](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/créer_une_snapshot_partie_2.png)  
Ajoutez (ou pas) une description, puis créez l'instantané. Vous pouvez vérifier sa création dans l'onglet "Instantanés" (sur la droite) :  
![Snapshot partie 3](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/créer_une_snapshot_partie_3.png)
### IPTables
Vous pouvez créer une nouvelle couche de Firewall avec le service iptables. Pour l'installer, entrez dans la machine :  
```
apt install iptables
```
Voici les commande pour entrer une règle dans iptables. La première pour un port spécifique, la deuxième pour un groupe de ports :  
```
iptables -A INPUT -p tcp --dport numéro_du_port -j ACCEPT
iptables -A INPUT -p tcp --match multiport --dports numéro_du_premier_port:numéro_du_dernier_port -j ACCEPT
```
Voilà à quoi ressemble le mien :  
![iptables](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/iptables_rules.png)
### Fail2Ban
Fail2Ban est une couche supplémentaire de protection, qui mettra en prison (jail) les IPs qui échouent à ce connecter de multiple fois au serveur. Vous pouvez l'installer avec :  
```
apt install fail2ban
```
Le fichier de configuration des prisons est /etc/fail2ban/jail.conf :  
![fail2ban](https://raw.githubusercontent.com/remymarchal/AWS/main/captures%20AWS/fichier_conf_fail2ban.png)