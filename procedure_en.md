# Procedure to set up a containerized WordPress site on Docker.
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

**uploads.ini
```ini
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
```

Start everything with `docker-compose up -d`
To stop: `docker-compose down`

### Features:
- Sets up a WordPress site with docker.
- Accessible on port 8080.
- Data is saved in the current folder.
- DB: SQL

## 2 - Integration with Swarm

Replace the docker-compose file with:
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
Add the following file at the root of the folder:
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
Now, to initialize the Swarm, enter the following command on the machine that will serve as host.
`docker swarm init --advertise-addr [HOST-IP]`
This command will give another command that will need to be used on the Worker machine to add it to this swarm. This command will take the following form:
`docker swarm join --token [TOKEN] [IP]`
After adding the Worker to the Swarm, enter the following command on the Host machine:
`docker stack deploy -c docker-compose.yml wordpress`
This will deploy the host configuration on the different workers, and start the service.