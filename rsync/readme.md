# Rsync Unauthorized Access
## Présentation du logiciel

Rsync (Remote Sync) est un outil de synchronisation de fichiers très rapide et polyvalent, largement utilisé sur les systèmes Linux/Unix pour les sauvegardes et le transfert de données. Il peut fonctionner via SSH ou via son propre démon (service) sur le port TCP 873.

## Chargement de l'image Vulhub
```
git clone https://github.com/vulhub/vulhub
cd rsync/common
sudo docker-compose up -d
```
## Présentation de la vulnérabilité

Il s'agit d'un accès non autorisé dû à une mauvaise configuration du démon Rsync. Dans cet environnement, le démon est configuré avec un module (un dossier partagé) accessible en lecture et écriture sans aucune authentification. Puisque le démon s'exécute souvent avec les privilèges root, cela permet à un attaquant de lire des fichiers sensibles ou d'écrire des scripts malveillants (comme des tâches Cron) pour obtenir une exécution de code à distance.

## Exploit de la vulnérabilité
```
rsync rsync://your-ip:873/src/
rsync -av rsync://192.168.125.128:873/src/etc/passwd ./
```
## Explication de la raison pour laquelle l’exploit fonctionne

L'exploit fonctionne parce que le fichier de configuration rsyncd.conf ne restreint pas l'accès.
En inspectant la configuration par défaut du conteneur, on constate que :
- Il n'y a pas de directive auth users définie pour le module [src].
- Il n'y a pas de directive secrets file pour vérifier les mots de passe.
- La directive read only = no est présente, autorisant l'écriture.
Par défaut, si aucune authentification n'est spécifiée, Rsync autorise les connexions anonymes. Comme le module path est défini sur / (la racine), tout le système de fichiers est exposé.

## Proposition justifiée d’un correctif



## Application du correctif
## Démonstration que l’exploit ne fonctionne plus
