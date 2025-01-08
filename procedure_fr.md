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

# Features:
- Monte un site wordpress avec docker.
- Accessible sur le port 8080.
- Les données sont enregistrées dans le dossier actuel.
- BDD: SQL