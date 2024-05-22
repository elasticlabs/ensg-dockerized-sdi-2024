---
{"dg-publish":true,"permalink":"/20-elastic-labs/202-e-labs-courses/22-ensg/221-sdi-with-microservices-2024-fr/labs/lab-02-introduction-a-docker-compose/","noteIcon":""}
---

Docker compose permet d'organiser les services, réseaux, ressources (configuration, données, etc.) d'une pile d'applications de manière déclarative. 
p
Commençons par un cas d'usage très simple : portainer. 

Créez un dossier dédié à vos travaux Docker dans votre répertoire personnel.

```
mkdir ~/Apps
```

Créez un réseau docker appelé ``revproxy_apps``
Créez un répertoire dédié à votre 1ère app, ou clonez votre dépôt github et dédiez un répertoire à votre 1ère app.

RDV sur https://github.com/docker/awesome-compose afin de chercher des points de départ fiables car produits par les équipes développant Docker. 

Cherchez l'exemple fourni pour portainer. : (rép. https://github.com/docker/awesome-compose/blob/master/portainer/compose.yaml) et intégrez le dans un fichier docker-compose.yaml

```
services:
  portainer:
    image: portainer/portainer-ce:alpine
    container_name: portainer
    command: -H unix:///var/run/docker.sock
    ports:
      - "9000:9000"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "portainer_data:/data"
    restart: always

volumes:
  portainer_data:
```

Tentez de construire la pile avec une commande `docker compose build`

- [?] Que se passe-t-il? pourquoi? 
- [?] Quel opérateur pour instancier portainer? 
- [?] Que faut-il ajouter pour la greffe prenne? 
- [?] Quel fichier final fonctionne-t-il? 

Instanciez la pile et connectez-vous à Portainer via le navigateur Firefox 

### Ajout de services

Avançons un peu dans la construction de notre IDG. 

Ajoutez maintenant un moteur de BDD et son interface d'administration : PostreSQL et Pgadmin. RDV sur https://github.com/docker/awesome-compose afin de chercher des points de départ fiables car produits par les équipes développant Docker. 

Ajoutez pgadmin et postgis à votre pile depuis les sources fournies dans ce dépôt. 

- [!] Vous allez devoir ajouter un fichier d'environnement pour compose. Plusieurs manières de le configurer : 
	- renseigner un fichier `.env` dans la racine du projet, contenant l'ensemble des variables. 
	- créer un répertoire spécifique à un COTS (e.g. postGIS) et référencer un fichier d'environnement, au niveau du `docker-compose.yaml` (opérateur `env_file` pour le fichier d'environnement)

Postgresql requiert à minima les variables suivantes : 

```
POSTGRES_USER=yourUser
POSTGRES_PW=changeit
POSTGRES_DB=postgres
PGADMIN_MAIL=your@email.com
PGADMIN_PW=changeit
```

Créez le fichier et modifiez ces variables à votre convenance, puis renseignez le fichier `docker-compose.yaml` avec les informations trouvées sur le dépôt GIT. 

- [?] Quels problèmes pose ce déploiement à notre cas d'usage? 
	- Du point de vue des accès au SI?
	- Du point de vue de la réponse au besoin?

### De postgres à PostGIS

Modifiez vos services afin de partir de la solution "préfabriquée" Postgres à une solution dérivée, plus spécialisée GEO : `postgis`

- [i] Recherchez une image officielle
- [i] Lisez sa documentation afin de clarifier : 
	- Versions de SGBDR attachées aux versions de conteneurs
	- Compatibilités avec votre pile
	- Besoins complémentaires en configuration
	- Indications de mise en oeuvre 

- [?] Quelle est la version la plus stable de `postgis`?
- [?] Quelle version est la plus difficile à étendre? 
- [?] Quelle est la différence entre les opérateurs `image` et `build`?

``` docker-compose.yaml
[...]

  postgis:
    container_name: ensg-sdi-postgis
    image: postgis/postgis:${POSTGIS_VERSION_TAG:-16-3.4}
    volumes:
      - postgis_data:/var/lib/postgresql/data
    ports:
      - 127.0.0.1:${POSTGIS_PORT_EXTERNAL:-8001}:5432
    environment:
      - "PGUSER=${POSTGRES_USER:-gis}"
      - "POSTGRES_USER=${POSTGRES_USER:-gis}"
      - "POSTGRES_DB=${POSTGRES_DB:-gis}"
      - "POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-gis}"
    shm_size: 256MB
```

OK, organisons maintenant un peu mieux notre projet afin un petit génie de l'automatisation : l'utilitaire `make`!
