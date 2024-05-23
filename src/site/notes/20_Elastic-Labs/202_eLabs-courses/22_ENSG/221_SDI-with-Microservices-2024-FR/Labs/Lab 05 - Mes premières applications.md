---
{"dg-publish":true,"permalink":"/20-elastic-labs/202-e-labs-courses/22-ensg/221-sdi-with-microservices-2024-fr/labs/lab-05-mes-premieres-applications/","noteIcon":""}
---


Le reverse proxy maintenant en place, attelons-nous à la composition du reste de l'IDG. Pour commencer, plaçons : 
- Pour le hub : une `page d'accueil`, un `gestionnaire de fichiers` et `portainer` 
- Pour la SDI :  `postgis` et son interface de gestion : `pgadmin`, ainsi que geoserver.  

### Etapes à suivre 
- Décider du plan d'adressage et de nommage
- Configurer chaque brique en accord au niveau du reverse proxy
- Configurer indépendamment chaque outil par rapport à vos besoins spécifiques

Pour rappel, le lab fonctionne sur un réseau arbitrairement créé pour ses besoins

```bash
docker network create --subnet=172.24.0.0/16 --driver bridge revproxy_apps
```

### Plan de nommage et adressage 
Chaque service devant faire l'objet d'une résolution DNS devra disposer d'une IP fixe dans le cadre d'un lab local. 
- [i] Les conteneurs docker situés sur un même réseau communiquent ensemble grâce à leur nom de conteneur spécifié dans compose; pratique! 😊

| Service     | Nommage                       | @IP?       |
| ----------- | ----------------------------- | ---------- |
| dns-gen     |                               | 172.24.0.2 |
| portainer   | hub.ensg-sdi.docker/portainer | 172.24.0.4 |
| homepage    | hub.ensg-sdi.docker/          | 172.24.0.5 |
| filebrowser | hub.ensg-sdi.docker/data      | 172.24.0.6 |

## Configuration de docker compose

### `docker-compose-hub.yml`

Nous allons ajouter 2 outils fort utiles à l'administration d'une IDG : 
- Une page d'accueil aux stéroïdes : [homepage](https://gethomepage.dev/latest/installation/docker/)
- Un gestionnaire de fichiers : [filebrowser](https://github.com/hurlenko/filebrowser-docker)

Editez le fichier `docker-compose-hub.yml` et ajoutez nos 2 services : 

```docker
  #
  # SDI Access hub
  homepage:
    image: ghcr.io/benphelps/homepage:latest
    container_name: ${COMPOSE_PROJECT_NAME}_homepage
    restart: unless-stopped
    depends_on:
      - nginx-proxy
    expose:
      - "3000"
    volumes:
      - ./data/homepage:/app/config
      - homepage-public:/app/public
    networks:
      elabs-revproxy:
        ipv4_address: 172.24.0.5
  
  #
  # File browser
  # See https://github.com/hurlenko/filebrowser-docker for more information
  filebrowser:
    image: hurlenko/filebrowser
    container_name: ${COMPOSE_PROJECT_NAME}_filebrowser
    restart: unless-stopped
    environment:
      - PUID=$(id -u)
      - PGID=$(id -g)
      - FB_BASEURL=/data
      # The list of avalable options can be found here : https://filebrowser.org/cli/filebrowser#options.
    expose:
      - "443:8080"
    volumes:
      - ./data/filebrowser/filebrowser.db:/database/filebrowser.db
      - ./data/filebrowser/config:/config/
      - ./data/geoserver/host-files:/data/geoserver-host-files
      - ./data/files:/data/host-files
    networks:
      elabs-revproxy:
        ipv4_address: 172.24.0.6
```

N'oubliez pas d'inventorier les volumes *nommés* dans la rubrique *volumes*!

### Configuration du reverse proxy

Commençons par la configuration de `homepage`
- [?] Où est-il déployé vis à vis du proxy? 

| Service     | Nommage                  | @IP?       |
| ----------- | ------------------------ | ---------- |
| homepage    | hub.ensg-sdi.docker/     | 172.24.0.5 |
| filebrowser | hub.ensg-sdi.docker/data | 172.24.0.6 |
Pour ce faire, modifiez le contenu du fichier `proxy-hub.conf`, contenant la configuration du proxy Nginx héritée du lab précédent.

```nginx 
		##
        # -> Homepage : admin dashboard for our SDI
        location / {
	        #include /etc/nginx/proxy.conf;
		    set $upstream_app homepage;
		    set $upstream_port 3000;
		    set $upstream_proto http;
		    proxy_pass $upstream_proto://$upstream_app:$upstream_port;
        }
        ##

		##
        # -> Filebrowser : files admin web GUI for our stack
        location /data {
            # prevents 502 bad gateway error
            proxy_buffers 8 32k;
            proxy_buffer_size 64k;
            client_max_body_size 75M;

			# redirect all HTTP traffic to localhost:8088;
            proxy_pass http://filebrowser:8080;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $http_host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            #proxy_set_header X-NginX-Proxy true;
            # enables WS support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_read_timeout 999999999;
        }
        [...]
```

### Configuration de l'outil `homepage`

Commençons par la configuration de `homepage`
- [?] Où est située sa configuration?

Localisez sa configuration et modifiez-la afin de l'adapter à vos besoins. Les fichiers suivants permettent de configurer l'outil : https://gethomepage.dev/latest/configs/

## Instanciation et accès aux outils! 

Les cibles et oérateurs `make` créés précédemment devraient suffire à couvrir ces besoins. Répétez le cycle et déboguez si besoin. 

**Adresses  des outils déployés à ce stade** : 

| Service     | Nommage                       |
| ----------- | ----------------------------- |
| portainer   | hub.ensg-sdi.docker/portainer |
| homepage    | hub.ensg-sdi.docker/          |
| filebrowser | hub.ensg-sdi.docker/data      |
