# Procedure to set up a containerized WordPress site on Docker

## 1 - WordPress Dockerization
- At the root of your project, create the following files:

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

**uploads.ini**
```ini
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
```

Start everything with `docker-compose up -d`
To stop: `docker-compose down`

# Features:
- Sets up a WordPress site with Docker
- Accessible on port 8080
- Data is saved in the current folder
- Database: SQL