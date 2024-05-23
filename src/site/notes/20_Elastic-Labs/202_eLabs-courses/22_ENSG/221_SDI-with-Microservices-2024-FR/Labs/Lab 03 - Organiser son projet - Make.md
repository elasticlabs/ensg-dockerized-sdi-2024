---
{"dg-publish":true,"permalink":"/20-elastic-labs/202-e-labs-courses/22-ensg/221-sdi-with-microservices-2024-fr/labs/lab-03-organiser-son-projet-make/","noteIcon":""}
---


Cloner le dépôt ``https://github.com/elasticlabs/sdi-mod1-reverse-proxy`` dans votre répertoire ``~/Apps``

Déplacez-vous dans le répertoire cloné, et tentez de démarrer la pile logicielle avec la commande : 

```
user@host# docker compose build
```

- [?] *Que se passe-t-il?* 
- [?] *Quelle explication?*

---
### Configurer l'environnement de docker compose

Parmi les fichiers que vous y trouverez, modifiez celui contenant l'environnement d'exécution de ``docker compose`` : le fichier ``.env-changeme``
- Modifiez le nom du projet. Par exemple : ``ensg-sdi-entrypoint``
- Modifiez le nom de domaine de votre SDI 
	- <u>En local</u> : un nom parlant et se terminant en ``.docker`` . Par exemple : ``ensg-sdi.docker`` 
	- <u>En ligne</u> : un nom préalablement enregistré dans le serveur DNS attaché à votre nom de domaine (par exemple Google domains)

```
# 1/ Project name
COMPOSE_PROJECT_NAME=changeme

# 2/ Proxy docker networks
#  - APPS_NETWORK will be the network name you use in deployed compose services
APPS_NETWORK=revproxy_apps
```

Une fois les changements effectués, copiez le fichier ``.env-changeme`` en un fichier ``.env`` dans le même répertoire. 

L'utilitaire `make` permet de regrouper des ensembles de commandes Shell dans des opérateurs, afin de préparer des tâches complexes et paramétriques, non prises en charge par les COTS directement. 
- Par exemple, `compose` orchestre l'instanciation d'un ensemble de ressources mais...
- [?] Que faire si nous devons tout de même modifier de nombreuses fois un paramètre (e.G. nom de domaine) dans la configuration des logiciels de la pile? 
- [?] Comment automatiser des tâches qui requièrent une certaine expérience même en présence de documentation? 

`make` dispose d'une syntaxe très proche du shell UNIX, et permet de regrouper TOUT ce que les autres ne font pas. Pratique. Efficace. Complémentaire. 

Commencez par créer un fichier `makefile` avec les informations suivantes (nota : les commandes Shell UNIX doivent être précédées par un `@` lors de leur appel) : 

```
# Set default no argument goal to help
.DEFAULT_GOAL := help

# Ensure that errors don't hide inside pipes
SHELL         = /bin/bash
.SHELLFLAGS   = -o pipefail -c

# Variables initialization from .env file
DC_PROJECT?=$(shell cat .env | grep COMPOSE_PROJECT_NAME | sed 's/^*=//')
APP_URL?=$(shell cat .env | grep VIRTUAL_HOST | sed 's/.*=//')
CURRENT_DIR?= $(shell pwd)

# Every command is a PHONY, to avoid file naming confliction -> strengh comes from good habits!
.PHONY: help
help:
    @echo "=============================================================================================="
    @echo "                Building a simple Bubble Apps proxy composition "
    @echo "             https://github.com/elasticlabs/elabs-bubble-apps-proxy"
    @echo " "
    @echo "Hints for developers:"
    @echo "  make up             # With working HTTPS proxy, bring up the Bubble stack"
    @echo "  make down           # Brings the Bubble stack down. "
    @echo "  make update         # Update the whole stack"
    @echo "  make cleanup   # Complete hard cleanup of images, containers, networks, volumes & data"
    @echo "=============================================================================================="

```

Notez que pour le moment, ce fichier ne semble rien apporter de particulier. 
Enregistrez-le, et lancez simplement al commande `make`

- [?] Qu'obtenez-vous?
- [?] Que se passe-t-il si vous tentez d'exécuter l'une des opérations décrites? 

### 1ères opérations `make`

Ecrivez maintenant un opérateur `make` dans le `makefile` permettant le *build* de la pile logicielle. 

```
[...]
.PHONY: build
build: pull
	@echo "[Make] Build the stack"
	docker-compose -f docker-compose.yml build

.PHONY: pull
pull:
	@echo "[Make] Pull the stack images"
    docker-compose -f docker-compose.yml pull
```

Tentez maintenant d'exécuter la commande `make build`? Qu'observez-vous? 

Ajoutez maintenant les opérateurs suivants : 
- `make up`
- `make update`

### Tâches de nettoyage

`Make` est également très utile en matière de nettoyages des piles logicielles. En effet, nombreuses sont les occasions de remettre en question ce que l'on a créé, et d'avoir besoin de remettre en question tout ou partie de notre travail. 

De plus, il est également nécessaire de nettoyer complètement notre environnement dans le cadre de travaux de consolidation de développements. 

```
.PHONY: hard-cleanup

hard-cleanup:
    @echo "[INFO] Bringing done the HTTPS automated proxy"
    docker compose -f docker-compose.yml down --remove-orphans
    # Delete all hosted persistent data available in volumes
    @echo "[INFO] Cleaning up static volumes"
    # Adjust the following command to fit your needs
    # docker volume rm -f $(PROJECT_NAME)_html
    @echo "[INFO] Cleaning up containers & images"
    docker system prune -a
```

- [?] Quelles sont les 4 opérations réalisées par cette tâche de nettoyage? 
	- Pile logicielle
	- Volumes
	- Quels sont éléments supprimés par la commande `docker system prune -a` ?

### Opérateur `up` et lancement de notre pile

Ajoutez maintenant l'opérateur permettant de démarrer notre pile logicielle : 

```
.PHONY: up
up: build
    @bash ./.utils/message.sh info "[INFO] Bringing up the stack"
    docker compose up -d --remove-orphans
```

- [?] Comment accédez-vous aux outils? 
- [?] Quels sont les problèmes à résoudre?
- [?] Quelles différences avec l'accès à des services sur l'Internet? 

