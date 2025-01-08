# Procédure pour mettre en place un site wordpress contenérisé sur Docker.
## 1 - Dockerisation de Wordpress
- A la racine de votre projet, créer les documents suivants:
**docker-compose.yml**
```yaml
version: '3.8'

services:
  # Database
  db:
    image: mysql:5.7
    volumes:
      - ./data/mysql:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    networks:
      - wordpress_network

  # WordPress
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    volumes:
      - ./data/wordpress:/var/www/html
      - ./config/uploads.ini:/usr/local/etc/php/conf.d/uploads.ini
    ports:
      - "8080:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    networks:
      - wordpress_network

networks:
  wordpress_network:
    driver: bridge
```

**uploads.ini
```ini
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
```

Démarrer le tout avec `docker-compose up -d`
Pour arréter: `docker-compose down`

### Features:
- Monte un site wordpress avec docker.
- Accessible sur le port 8080.
- Les données sont enregistrées dans le dossier actuel.
- BDD: SQL

## 2 - Intégration avec Swarm

Remplacer le fichier docker-composer par:
**docker-compose.yml**
```yaml
version: '3.8'

services:
  db:
    image: mysql:5.7
    volumes:
      - mysql_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    networks:
      - wordpress_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
          
  wordpress:
    image: wordpress:latest
    volumes:
      - wordpress_data:/var/www/html
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    networks:
      - wordpress_network
    deploy:
      mode: replicated
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      
  loadbalancer:
    image: nginx:latest
    ports:
      - "80:80"
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
    depends_on:
      - wordpress
    networks:
      - wordpress_network
    deploy:
      mode: replicated
      replicas: 2

configs:
  nginx_config:
    file: ./nginx.conf

volumes:
  mysql_data:
    driver: local
  wordpress_data:
    driver: local

networks:
  wordpress_network:
    driver: overlay
```
Ajouter le fichier suivant à la racine du dossier:
**nginx.conf**
```conf
events {
    worker_connections 1024;
}

http {
    upstream wordpress {
        server wordpress:80;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://wordpress;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```
Maintenant, pour initialiser la Swarm, entrer la commande suivante sur la machine qui servira d'hôte.
`docker swarm init --advertise-addr [IP-DE-L'HOTE]`
Cette commande donnera une autre commande qu'il faudrat utiliser sur la machine Worker pour l'ajouter à cette swarm. Cette commande prendra la forme suivante:
`docker swarm join --token [TOKEN] [IP]`
Après avoir ajouté le Worker à la Swarm, entrer la commande suivante sur la machine Hôte:
`docker stack deploy -c docker-compose.yml wordpress`
Celle-ci vas deployer la configuration de l'hôte sur les différents workers, et démarrer le service.