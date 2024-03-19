---
{"dg-publish":true,"permalink":"/20-elastic-labs/202-e-labs-courses/22-ensg/221-sdi-with-microservices-2024-fr/labs/lab-04-mise-en-place-d-un-reverse-proxy/","noteIcon":""}
---



Un reverse proxy permet d'offrir à votre système d'informations une interface principale, positionnée en frontal de la plupart des échanges au sein du SI. 
- Gestion des entrées / sorties des flux utilisateurs
- Gestion du nommage des services
- Gestion des utilisateurs
- Centralisation des besoins de monitoring
- Centralisation des besoins de sécurité

Pour les besoins de ce lab, commençons d'abord par la mise en place d'une zone d'administration contenant notre proxy, et quelques services sidecar : 
- **Portainer** : administration de l'environnement docker
- **DNSMasq** : Serveur de nommage automatisé de notre SI local (a retirer une fois en ligne (sera vu jour 4)

Une fois ce lab réalisé, vous devriez pouvoir réaliser les opérations suivantes :

| Opération              | Détails                                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------------------ |
| Résolution de noms DNS | Domaine `hub.ensg-sdi.docker`                                                                                |
| Accès à portainer      | Portainer accessible via firefox à l'adresse : `http://hub.ensg-sdi.docker/portainer`                        |
| Makefile initialisé    | Cibles de déploiement complètement outillées : <br>-   Préparation de l'environnement<br>-   Opérations CRUD |
Clonez le dépôt ``https://github.com/elasticlabs/sdi-mod2-reverse-proxy`` dans votre répertoire ``~/Apps``

### Composition de la pile proxy

Editez le fichier docker-compose.yml et insérez le bloc suivant afin de créer un serveur proxy basé sur NGinx : 

```docker
version: '3'

services:
  #
  # --> Reverse proxy
  nginx-proxy:
    image: ${COMPOSE_PROJECT_NAME}_proxy:latest
    container_name: ${COMPOSE_PROJECT_NAME}_proxy
    restart: unless-stopped
    expose:
      - "80"
    build:
      context: ./data/nginx-proxy
    environment:
      - DHPARAM_GENERATION=false
      - VIRTUAL_PORT=80
      - VIRTUAL_HOST=hub.ensg-sdi.docker
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
```

En bas du fichier docker-compose.yml, vous repérez 2 blocs : 
- Volumes
- Networks

Pour les besoins de notre lab, le réseau elabs-revproxy a déjà été créé avec les paramètres suivants : 
- [?] Que signifient ces paramètres?
- [?] Quelles implications pour notre configuration?

```docker
  [...]
networks:
  elabs-revproxy:
    name: ${APPS_NETWORK}
    external: true
    ipam:
      config:
        - subnet: "172.24.0.0/16"
```

Rattachez maintenant votre service nginx-proxy à ce réseau, et attribuez-lui une adresse IP. Ce besoin est spécifique à notre labo local. ==Pourquoi?==

```docker
services:
  #
  # --> Reverse proxy
  nginx-proxy:
  [...]
  networks:
    elabs-revproxy:
      ipv4_address: 172.24.0.3
```

### Configuration de l'image de notre proxy

SI vous regardez de plus près ce fichier docker-compose.yml pré renseigné, vous apercevez les lignes suivantes : 

```docker
build:
  context: ./data/nginx-proxy
```

- [?] Quelle est leur signification? 
- [?] Quelle est la prochaine étape de configuration du proxy?

## Configuraiton de l'image `nginx-proxy`

Déplacez-vous dans le répertoire de l'application, et créez : 

```bash 
cd ~/Apps/sdi-mod2-reverse-proxy/data/
mkdir nginx-proxy
```

Créez un fichier `Dockerfile` décrivant la construction de l'image du proxy : 

```shell 
cd data/nginx-proxy
nano Dockerfile
```

Renseignez le contenu suivant : 

```docker
FROM nginx:alpine

COPY proxy.conf /etc/nginx/proxy.conf
COPY uploadsize.conf /etc/nginx/conf.d/uploadsize.conf
COPY proxy-hub.conf /etc/nginx/conf.d/proxy-hub.conf
```

Une image très simple, en somme. Reste à créer les fichiers copiés. 

Créez le fichier `proxy.conf` initialisé avec le contenu suivant : 
- [i] Il contient les paramètre généraux de notre serveur proxy

```nginx
# HTTP 1.1 support
proxy_http_version 1.1;
proxy_buffering off;
proxy_set_header Host $http_host;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $proxy_connection;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $proxy_x_forwarded_proto;
proxy_set_header X-Forwarded-Ssl $proxy_x_forwarded_ssl;
proxy_set_header X-Forwarded-Port $proxy_x_forwarded_port;

# Mitigate httpoxy attack
proxy_set_header Proxy "";

# Custom Values Higher Buffer Size
proxy_buffer_size          128k;
proxy_buffers              4 256k;
proxy_busy_buffers_size    256k;
```

Créez ensuite le fichier suivant : `uploadsize.conf`
- [i] Il contient des paramètres évitant certaines déconvenues lors de transferts de fichiers

```nginx
client_max_body_size 512M;
large_client_header_buffers 4 32k;
```

Créez enfin le fichier `proxy-hub.conf`, contenant un début de configuration du proxy Nginx .

```nginx
log_format vhost '$host $remote_addr - $remote_user [$time_local] '
                 '"$request" $status $body_bytes_sent '
                 '"$http_referer" "$http_user_agent"';
#
# Set resolver to docker default DNS
resolver 127.0.0.11 valid=30s;

# server blocks definition
server {
        server_name hub.ensg-sdi.docker;
        listen 80 ;
        access_log /var/log/nginx/access.log vhost;
        #
        # -> We store DSI apps in a separate file
        #include /etc/nginx/conf.d/sdi-apps.conf;
 
        ##
        # -> Portainer
        location /portainer {
            return 301 $scheme://$host/portainer/;
        }

        location ^~ /portainer/ {
            # enable for Authelia (requires authelia-server.conf in the server block)
            #include /config/nginx/snippets/authelia-authrequest.conf;
  
            include /etc/nginx/proxy.conf;
            # include /config/nginx/resolver.conf;
            set $upstream_app portainer;
            set $upstream_port 9000;
            set $upstream_proto http;
            proxy_pass $upstream_proto://$upstream_app:$upstream_port;
  
            rewrite /portainer(.*) $1 break;
        proxy_hide_header X-Frame-Options; 
        }
  
        location ^~ /portainer/api {
            include /etc/nginx/proxy.conf;
            # include /config/nginx/resolver.conf;
            set $upstream_app portainer;
            set $upstream_port 9000;
            set $upstream_proto http;
            proxy_pass $upstream_proto://$upstream_app:$upstream_port;
  
            rewrite /portainer(.*) $1 break;
            proxy_hide_header X-Frame-Options; # Possibly not needed after Portainer 1.20.0
        }
        ##

    }
```

## Composition de la pile logicielle du proxy

### dnsmasq
Revenez dans le répertoire racine de votre composition, et ajoutez désormais le service `dns-gen` :
- [i] Il s'agit du serveur DNS automatisé construit pour les besoins de ce lab. Basé sur les COTS `dnsmasq` et `docker-gen`, il réalise les opérations suivantes comme on le ferait à la main sur Internet : 
	- Ecoute des opérations CRUD docker sur le fichier socket docker
	- Génération d'un nom DNS pour tout conteneur disposant de la variable VIRTUAL_HOST renseignée)
		- [i] Domaine racine choisi pour ce lab : `.docker`
	- Création de noms DNS automatisés pour chaque conteneur : 
		- service.projet_compose.docker (*e.g. nginx-proxy.ensg-sdi-2024.docker*)

```docker
[...] 

services:
  [...]
  #
  # dnsmasq config for containers
  dns-gen:
    build: ./data/dns-gen
    container_name: ${COMPOSE_PROJECT_NAME}_dns-gen
    restart: unless-stopped
    expose:
      - "53:53/udp"
    depends_on:
      - nginx-proxy
    logging:
      options:
        max-size: "10m"
    volumes:
      - dns-gen-config:/etc/
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      elabs-revproxy:
        ipv4_address: 172.24.0.2

volumes: 
  - dns-gen-config:
[...]
```

Remarquez l'ajout d'un volume en bas du fichier. Il s'agit d'un volume nommé utilisé par le service dns-gen, et géré en totale autonomie par docker.

### Portainer

Bien. Ajoutez enfin un bloc dédié au service portainer pour parfaire cette 1ère pile brique de notre hub d'administration.

```docker
[...]
services: 
  [...]
  #
  # Portainer docker admin GUI
  portainer:
    image: portainer/portainer-ce:latest
    container_name: ${COMPOSE_PROJECT_NAME}_portainer
    restart: unless-stopped
    depends_on:
      - nginx-proxy
    expose:
      - "9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data
    networks:
      elabs-revproxy:
        ipv4_address: 172.24.0.4
  [...]
volumes: 
  - dns-gen-config:
  - portainer-data:
[...]
```

### 1ère instanciation de la pile du proxy

Tentez de construire et instancier la pile du proxy avec les commandes suivantes : 
```
docker compose -f docker-compose-hub.yml --build
[...]
# En bas de réussite uniquement : 
docker compose -f docker-compose-hub.yml up -d --build
```

- [?] Quelle commande docker permet-elle d'observer l'état de démarrage des services / conteneurs? 
- [?] Quelle commande docker-compose permet-elle de tracer ce qu'il se passe à l'intérieur des conteneurs?

## Accès aux services

Tentez d'abord d'accéder à `nginx` et à `portainer` indépendamment afin de valider leur comportement. 

Tentez maintenant d'accéder à portainer directement via Firefow, à l'aide de l'URL `ensg-sdi.docker/portainer`
- [?] Quel comportement observez-vous? 
- [?] Comment pouvez-vous expliquer ce phénomène, et comment le résoudre?

## Configuration de la résolution de noms DNS

La résolution de noms DNS est un sujet crucial sur l'internet. Au sein d'un système UNIX, les fichiers suivants permettent de jeter un oeil à la configuration courante : 
- `/etc/hosts`
- `/etc/resolv.conf`
	- [?] De quelle nature est ce fichier? Quelle conclusion en retirer?

### Installation de resolvconf

Le fichier resolv.conf est géré d'une manière ambigüe et conjointe par plusieurs processus bas-niveau. Le fichier `/etc/resolv.conf` est de fait volatile, ce qui rend compliquée la modification pérenne des paramètres de résolution de nom. 
- [!] L'installation d'un utilitaire dédié permet de s'affranchir du problème
	- [i] Plus d'infos? https://www.tecmint.com/set-permanent-dns-nameservers-in-ubuntu-debian/

Installez l'utilitaire resolvconf à l'aide des commandes suivantes : 

``` bash
$ sudo apt update
$ sudo apt install resolvconf
```

Vérifiez qu'il est correctement installé et activé à l'aide des commandes suivantes : 

```bash
$ sudo systemctl start resolvconf.service
$ sudo systemctl enable resolvconf.service
$ sudo systemctl status resolvconf.service
```

![Pasted image 20240317172818.png](/img/user/20_Elastic-Labs/202_eLabs-courses/22_ENSG/221_SDI-with-Microservices-2024-FR/_imgs/Pasted%20image%2020240317172818.png)

Ajoutez maintenant notre serveur DNS local en entête des serveurs de DNS de référence sur la machine : 

```bash
$ sudo nano /etc/resolvconf/resolv.conf.d/head

Ajoutez la ligne suivante : 

nameserver 172.24.0.2
```

Sauvegardez le fichier, et redémarrez successivement : 

```bash
$ sudo systemctl restart resolvconf.service
$ sudo systemctl restart systemd-resolved.service
```

Vérifiez que la ligne `nameserver 172.24.0.2` est bien présente dans le fichier `/etc/resolv.conf`; si elle est encore absente, redémarrez la station de travail puis revérifiez.

- [!] Il arrive que malgré tout, ces étapes ne suffisent pas. Le cas échéant, saisissez la commande suivante afin de reconfigurer `resolvconf` : 

```bash
$ sudo dpkg-reconfigure resolvconf
```

... Puis redémarrez et admirez le résultat! 

## Configuration des cibles `make`

Les commandes de construction et instanciation de la pile logicielle vues ensemble plus haut peuvent maintenant être directement insérées dans votre makefile, afin de ne plus avoir besoin de les saisir à nouveau. 

Au passage, profitons-en pour créer notre réseau `elabs-revproxy` s'il n'existe pas ou plus (comme après un nettoyage de fond, par exemple)

Editez le fichier `makefile` prérempli de la composition du reverse proxy.  

```makefile
[...]
    @echo "  make cleanup      # /!\ Remove images, containers, volumes & data"
    @echo "  make update           # Update the whole stack"
    @echo "======================================================================="

.PHONY: hub-build
hub-build:
    # Network creation if not done yet
    @echo "[INFO] Create ${APPS_NETWORK} network if doesn't already exist"
    docker network inspect ${APPS_NETWORK} >/dev/null 2>&1 || docker network create --subnet=172.24.0.0/16 --driver bridge ${APPS_NETWORK}
    #
    # Build the stack
    @echo "[INFO] Building the stack"
    docker compose -f docker-compose-hub.yml build
    @echo "[INFO] Build OK."

.PHONY: hub-up
hub-up: build
    @echo "[INFO] Bringing up the Hub"
    docker compose -f docker-compose.yml up -d --remove-orphans

.PHONY: hub-set-hosts
hub-set-hosts:
    @echo "[INFO] Updating system hosts file (sudo mode)"
    sudo cp ${DNSMASQ_CONFIG}/hosts.dnsmasq /etc/hosts
```

- [?] Comment le nom du réseau est-il récupé automatiquement par Make? 
- [?] Que se passe-t-il si l'on lance la commande `make hub-set-hosts`? 

### Instanciez votre proxy! 

Vous pouvez désormais exécuter les commandes suivantes afin de mettre en oeuvre votre reverse proxy : 
- `make hub-build`
- `make hub-up`

Une fois ce lab réalisé, vous devriez pouvoir réaliser les opérations suivantes :

| Opération              | Détails                                                                               |
| ---------------------- | ------------------------------------------------------------------------------------- |
| Résolution de noms DNS | Domaine `hub.ensg-sdi.docker`                                                         |
| Accès à portainer      | Portainer accessible via firefox à l'adresse : `http://hub.ensg-sdi.docker/portainer` |
### Tâches utilitaires 

Ajoutons au fichier makefile les tâches "utilitaires" suivantes : 
- [i] Mises à jour de la pile logicielle
- [i] Nettoyage des assets docker et compose

Editez le fichier makefile, et ajoutez les opérateurs suivants permettant la mise à jour de votre pile logicielle : 

```make

.PHONY: update
update: pull hub-up wait
    docker image prune

.PHONY: pull
pull:
    docker compose -f docker-compose-hub.yml pull

.PHONY: wait
wait:
    sleep 10
```

Enfin, passons aux inévitables tâches de nettoyage de l'environnement docker :

```
.PHONY: cleanup
cleanup:
    @echo "[INFO] Bringing down the proxy HUB"
    docker compose -f docker-compose-hub.yml down --remove-orphans
    @echo "[INFO] Bringing down the SDI stack"
    docker compose -f docker-compose-hub.yml down --remove-orphans
    # Delete all hosted persistent data available in volumes
    @echo "[INFO] Cleaning up static volumes"
    #docker volume rm -f $(PROJECT_NAME)_ssl-certs
    #docker volume rm -f $(PROJECT_NAME)_portainer-data
    @echo "[INFO] Cleaning up containers & images"
    docker system prune -a
```
## Démarrage du hub et accès à portainer

Il ne vous reste plus qu'à instancier pour de bon cette porte d'entrée de votre SI, et vérifier que tout se passe bien. 

- [?] De votre point de vue, qu'est-ce qui manque encore à une expérience proche de celle disponible sur l'Internet?



