# Cheatsheet adminsys ElasticSearch

## ES, c'est quoi, comment ça marche ? 

Elasticsearch est un serveur d'indexation et de recherche sur de grands ensembles de données, qui utilise la plateforme JAVA et la librairie Lucene pour indexer et chercher dans du texte. 

* La clusterisation est native: un canal privilégié permet aux machines d'échanger entre elles.
* L'interface HTTP REST est utilisée pour s'adresser au serveur, pour les requêtes comme pour l'administration.
* Le format JSON en entrée/sortie rend ES compatible avec tous les langages de programmation.

### La clusterisation

Un **cluster** ES est constitué de plusieurs instances, dont chacune est appelée un noeud / **node**. Pour autant, il est possible de faire tourner un cluster ES avec un seul node.

Le Cluster se constitue dynamiquement, chaque node recherchant un cluster à rejoindre. Multicast est utilisé par défaut par les nodes pour trouver les membres de leur cluster. Mais il est possible pointer des adresses de serveurs individuellement en unicast.

### Réseau et sécurité du cluster

Par défaut, ES est ouvert au niveau réseau sans couche d'authentification. 

Si c'est acceptable pour un cluster dans un LAN/VLAN, c'est dangereux si les nodes ont des IP publiques.

Les adresses IP des nodes sont configurables séparément pour l'écoute du service (ex: 127.0.0.1, 0.0.0.0, 10.1.1.43) et pour l'intercommunication entre serveurs.

Configurer les adresses au plus juste et utiliser un firewall pour restreindre les accés sont de bonnes pratiques.

### Les rôles dans le cluster

Un cluster ES fonctionne avec des nodes ayant un ou plusieurs rôles : 
* Master : le node participe à l'élection (quorum) du node Master, qui décide de la validité des données et des nodes dans le cluster.
* Data : le node stocke localement des données du cluster
* Client/Ingest : le node reçoit et traite (tranforme) les requêtes utilisateurs 

### Etablissement du quorum

Un cluster ES constitué d'un seul node a tous les rôles : en cas de coupure, il est sa propre référence de données.

En cas de perte du master courant, les masters potentiels en élisent un nouveau. En cas de conflit, le groupe de masters le plus nombreux remporte l'élection. À 2 contre 1, impossible de tomber sur un score nul. 

Pour un cluster ES plus complexe, il est conseillé d'avoir trois serveurs avec le rôle Master pour éviter un effet de "Split Brain" en cas de perte d'une partie du cluster. 

Pour les clusters intensifs, il est conseillé d'avoir des masters dédiés : n'hébergeant ni données, ne recevant pas de requêtes, ils sont plus efficaces. 

### Les ports des nodes

Par défault, les nodes écoutent sur deux ports, configurables:  
* 9200 : port d'adressage des requêtes HTTP REST
* 9300 : port d'intercommunication entre machines du cluster

### Fonctionnement par API 

L'API ES est RESTful : elle utilise les méthodes HTTP GET, POST, PUT, DELETE, etc. pour passer des requêtes au cluster et lire / créer / remplacer / détruire / etc. des ressources. 

Par défaut, les requêtes sont gérées par le cluster dans son ensemble, quelque soit le noeud du cluster sur lequel on émet la requête.

#### Reprendre les exemples de la doc 

La documentation d'ES donne des exemples en éludant le protocole, l'adresse, et les arguments de CURL. Ainsi 

    PUT /_cluster/settings {
      "transient": {
          "cluster.routing.allocation.exclude._name": "myhost"
      }
    }

peut être exécuté comme ceci :

    curl -XPUT http://localhost:9200/_cluster/settings?pretty -d '{
      "transient": {
          "cluster.routing.allocation.exclude._name": "myhost"
      }
    }'

## La Réplication 
La répartition des données entre nodes dans un cluster ES dérive de la manière dont sont construites ces données. 


L'**index** (indice) est l'unité logique définie et utilisée par l'humain, c'est une collection de documents indexées interrogeable.
Un index est constitués de N éclats (shards), définis à la création de l'index.

Un **shard** est l'unité de stockage atomique, éventuellement répliquée.
0, 1 ou plusieurs replica (répliques) d'un shard primaire (primary) sont possibles
En cas de perte du primary shard, un réplica shard est promu comme nouvel éclat primaire. 

## Plugins
Des plugins web sont disponibles pour ES, en particulier pour afficher l'état du cluster et mieux l'administrer.
C'est particulièrement utile pour visualiser l'état du cluster ou la répartition des index et des shards.

On utilise l'exécutable $ES_HOME/bin/plugin pour ajouter des plugins à une instance. 

Le plugin est ensuite disponible sur le port 9200 du node, ex: http://localhost:9200/plugin/=plugin_name=. 

Quelques plugins utiles pour administrer son cluster :
* head
* sense
* elastiqsearch-hq
* cerebro
* bigdesk

## Configurer ElasticSearch

### Generalités
Le fichier de configuration principal est `/etc/elasticsearch/elasticsearch.yml`

La configuration est modifiable via l'API pour la partie Cluster :
* `transient` : le changement de configuration est annulé au redémarrage du cluster
* `persistent` : le changement de configuration est permanent

**ATTENTION, les changements "persistent" sont prioritaires par rapport aux fichiers de configuration ! **

Parmi les nombreux paramètres :  
* `node_name` : détermine le nom du node dans le cluster
* `bind_host` : détermine l'adresse IP sur laquelle écouteront les ports 9200 et 9300

### Logs
Le fichier de configuration `/etc/elasticsearch/logging.yml` contient les emplacements, filtres et durée de vie (rotation) des logs

Pour activer le log des requêtes lentes, il faut le préciser dans `elasticsearch.yml`

### Backups

Comme pour tout stockage de données, il est nécessaire de penser à la mise en place de backups et au test régulier de leur restoration.

Les backups ES sont réalisés sur toutes les machines du cluster, il faut donc utiliser un montage de fichier réseau. 

La définition des **repo** (points de montage) pour ce partage réseau est dans `elasticsearch.yml`. Pour un montage local à la NFS:
`path.repo: ["/mount/backups", "/mount/longterm_backups"]`

Pour lancer des backups et les restauration, on utilise l'API (voir plus loin).

### Garbage Collector

Le garbage collector de la machine java sous-jacente est responsable de la purge des adresses mémoire inutilisées. 

Il est conseillé de modifier les paramètres par défaut de la machine JAVA ES :
* copier le fichier `/usr/share/elasticsearch/bin/elasticsearch.in.sh` en `/usr/local/share/elasticsearch.elasticsearch.in.sh`
* trouver les options `-XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly`
* les remplacer par `-XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:InitiatingHeapOccupancyPercent=35`


# Commandes utiles pour administrer un cluster ElasticSearch

On utilise ici essentiellement les API CAT, Cluster, Indices

* L'API CAT affiche des informations lisibles par un humain sur l'état du cluster et des données. L'argument "?v" affiche les colonnes des tables.
* L'API Cluster affiche les informations du cluster en JSON. L'argument "?pretty" affiche le JSON avec des indentations.
* L'API Indices affiche les informations des index en JSON. L'argument "?pretty" affiche le JSON avec des indentations.

## Diagnostiquer 

### Voir l'état du cluster

    curl 'http://localhost:9200/_cat/health?v'

    ou curl 'http://localhost:9200/_cluster/health?pretty=true


Cette commande affiche en particulier les informations suivantes :
* status : l'état du cluster
    * green : tous les shards sont accessibles
    * yellow : tous les shards primaires sont accessibles, mais il manque des répliques
    * red : tout ou partie des shards primaires sont inaccessibles
* active_primary_shards : le nombre de shards primaires actifs
* active_shards : le nombre total de shards actifs, y compris les répliques
* relocating_shards : le nombre de shards en cours de déplacement
* unassigned_shards : le nombre de shards non disponibles 
* delayed_unassigned_shards : le nombre de shards non disponibles que le cluster ne va pas essayer de réallouer dans l'immédiat

### Lister les méthodes d'admin 

    curl 'http://localhost:9200/_cat?pretty=true'


### Lister les index

    curl 'http://localhost:9200/_cat/indices?pretty=true'


### Lister les noeuds

    curl http://localhost:9200/_cat/nodes?v


Pour choisir les colonnes

    curl http://localhost:9200/_cat/nodes?v\&pretty\&h=host,ip,version,jdk,heap.percent,ram.percent,load,node.role,master,name


Voir la documentation pour la liste des 80 colonnes affichables 

### Voir les allocations de shards et disque

    curl 'http://localhost:9200/_cat/allocation?v'


### Voir les informations sur les index

    curl 'http://localhost:9200/_cat/indices?v&health=yellow'


### Voir l'activité des nodes du cluster

    curl 'http://localhost:9200/_cat/thread_pool'


### Voir l'ensemble des stats

    curl 'http://localhost:9200/_cluster/stats?pretty'


## Gestion des index et shards

### Voir l'état de réallocation des shards

    curl 'http://localhost:9200/_cat/recovery'


### Voir la répartition des shards par node

    curl 'http://localhost:9200/_cat/shards?v'


### Déplacer un shard d'une machine à une autre

    curl -XPOST http://localhost:9200/_cluster/reroute -d '

      {

        "commands": [

          {

            "move": {

              "index": "export828",

              "shard": 0,

              "from_node": "bdd-01b",

              "to_node": "sibdd5"

            }

          }

        ]

      }'


### Changer le nombre de répliques d'un index

    curl -XPUT http://localhost:9200/<index name>/_settings?pretty -d '

      {

        "index": {

            "number_of_replicas": 0

        }

      }'


### Copier un index d'un cluster à un autre

    git clone https://github.com/geronime/es-reindex.gitcd es-reindex

    bundle install --path=gems

    bundle exec ./es-reindex.rb -h


## Gestion des nodes

### Vider un nœud de stockage
Utile pour décommissionner un nœud sans perte de redondance. 

    curl -XPUT http://localhost:9200/_cluster/settings?pretty -d '

      {

        "transient": {

            "cluster.routing.allocation.exclude._name": "bdd-01b"

        }

      }'

  
## Gestion de l'activité 

### Afficher le tâches en attente

    curl 'http://localhost:9200/_cat/pending_tasks?v'

  
## Gestion des Plugins

### Lister les plugins  

    curl 'http://localhost:9200/_cat/plugins?v'


## Gestion des backups

### Initialiser un espace de stockage 

	curl -XPUT 'http://localhost:9200/_snapshot/my_backup' -d '{
		"type": "fs",
		"settings": {
			"location": "/mount/backups/my_backup",
			"compress": true
		}
	}'

### Lancer un backup 

    curl -XPUT 'http://localhost:9200/_snapshot/my_backup/snapshot_1?wait_for_completion=true'

### Afficher les backups d'un espace de stockage

    curl 'http://localhost:9200/_snapshot/my_backup/_all'

### Restaurer un backup

    curl -XPOST 'http://localhost:9200/_snapshot/my_backup/snapshot_1/_restore'
    
### Supprimer un backup

    curl -XDELETE 'http://localhost:9200/_snapshot/my_backup'

## Sources

Cluster APIs
https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.html

CAT APIs
https://www.elastic.co/guide/en/elasticsearch/reference/current/cat.html

Indices APIs
https://www.elastic.co/guide/en/elasticsearch/reference/current/indices.html

Good Cheatsheets
https://blog.frankel.ch/elasticsearch-api-cheatsheet/
https://sematext.com/resources/elasticsearch-devops-cheat-sheet/
https://www.slideshare.net/fdevillamil/elasticsearch-at-synthesio






