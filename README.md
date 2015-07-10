# nginx-meteor
Docker image running nginx and meteor with embedded version of nodejs.

Use of embedded nodejs from meteor was taken from [grigio/docker-meteor](https://github.com/grigio/docker-meteor).

A barebones image with the only enforcement being nodejs running on port 3000 and everything else is up to you. The project specific configurations of meteor whether in dev or prod are pushed to docker-compose. An example project can be found at [mxs/todos-example-docker](https://github.com/mxs/todos-example-docker)

## dev

The dev environment should be as close as possible to the production one, hence moving mongo out to its own container.

Note the volume mapping of the source files and nginx configurations.

### docker-compose-dev.yml

```
todos:
  image: nginx-meteor
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./nginx/conf.d:/etc/nginx/conf.d
    - ./nginx/ssl:/etc/nginx/ssl
    - .:/meteorsrc
  environment:
    - MONGO_URL=mongodb://mongo:27017/test
  links:
    - mongo
  working_dir: /meteorsrc
  command: /bin/sh dev.sh

mongo:
   image: mongo
   volumes_from:
     - dbdata

dbdata:
   image: busybox
   volumes:
     - /data/db

```

### dev.sh
Starts up nginx and meteor in development mode.

```
service nginx start
meteor

```

## prod
### docker-compose.yml
The only thing different here from dev is the ROOT_URL environment variable must be set.

```
todos:
  image: nginx-meteor
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./nginx/conf.d:/etc/nginx/conf.d
    - ./nginx/ssl:/etc/nginx/ssl
    - .:/meteorsrc
  environment:
    - MONGO_URL=mongodb://mongo:27017/test
    - ROOT_URL=http://todos.net
  links:
    - mongo
  working_dir: /meteorsrc
  command: /bin/sh prod.sh

mongo:
   image: mongo
   volumes_from:
     - dbdata

dbdata:
   image: busybox
   volumes:
     - /data/db

```
### prod.sh

Here `meteor build` is used for production and supervisord for managing nginx and nodejs processes.

```
#building meteor and node package dependencies
rm -rf bundle
meteor build --directory .
(cd bundle/programs/server && npm install)

# Let supervisord run nginx and nodejs
/usr/bin/supervisord -c supervisor/todos.conf

```

### supervisor/todos.conf
Runs nginx in 'foreground' node for supervisor and piping logs to /dev/stdout so they can be viewed by `docker logs`.

```
[supervisord]
logfile=/dev/null
pidfile=/var/run/supervisord.pid
nodaemon=true

[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0

[program:nodejs]
command=/usr/bin/node /meteorsrc/bundle/main.js
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0

```
