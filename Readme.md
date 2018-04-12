# Traefik Docker Development

This service is a nginx reverse proxy like with its own strengh/advantages.<br />
I use it locally to work with docker and avoid port issues.<br />

The install is simple, once you have fetch this repository, use the following command to run it:<br />
```bash
docker-compose up -d
```

And you'are down. Traefik is now listening on the docker socket.

## Connect your docker web project for example

That's really simple, on your project, you may have a `docker-compose.yml` file ready to use.
To tell traefik you want to proxy your service, you have requirements as:
- Do not use the  `ports: - XXX:XXX` option to bind your local machine port, except if you want to access them separatly.
- Expose them using the `expose` settings
- Use labels to give directive to traefik
- Create a network called `traefik`: `docker network create traefik`

Example with nginx / phpfpm workflow
```yml
  # My app doesn't need to bind the port locally but need to be connected to the default network
  app:
    container_name: ${CONTAINER_PREFIX}.app
    working_dir: /app
    build: ./docker/app
    volumes:
      - .:/app
    expose:
      - 9000
    networks:
      - default

  # As it's our web access, we need to expose the port we need (could be also 80) but I use https)
  nginx:
    container_name: ${CONTAINER_PREFIX}.nginx
    working_dir: /app
    image: nginx:latest
    volumes:
      - .:/app
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ./docker/nginx/ssl:/etc/nginx/ssl
    depends_on:
      - app
    expose:
      - 443
    # don't forget to connect to traefik and to your default (named) network
    networks:
      - traefik
      - default  
    labels:
      - "traefik.backend=xi.front"
      - "traefik.docker.network=traefik"
      - "traefik.frontend.rule=Host:${APP_DOMAIN}"
      - "traefik.enable=true"
      - "traefik.frontend.entryPoints=https"
      - "traefik.port=443"
      - "traefik.protocol=https"

networks:
  traefik:
    external: true
```

So simply to give you what's going on here:
- `traefik.backend`, is the backend name for traefik where all the service will be put
- `traefik.docker.network`, We need to tell on which network we connect the service (the same under traefik)
- `traefik.frontend.rule`, Rule can contains more, but for our case you put the Host domain information (ex: mysuperapp.local)
- `traefik.enable=true`, Important if we want traefik to use the service, if you forget or set false, the service won't be mounted on traefik.
- `traefik.frontend.entryPoints`, Optional, by default it's `http` and `https` but for this case, I just want https
- `traefik.port`, Optionnal default is `80` but I need 443 for my https network
- `traefik.protocol`, Optional default is `http` but same, I want SSL.

Don't forget to add the external traefik network to connect your service.

Last things you may noticed, you have to add a new entry to your `/etc/hosts`, ex:
```
127.0.0.1 mysuperapp.local 
```

Now if you request this domain it will be forward to your localhost and traefik will forward this one to the approriate service listening on this domain.

You also can forward any service, port as a database, redis, messaging queue using a classic 80 port for traefik but behing it listens on the default one.

# Trafik settings

By default, it binds and listens on port 80, 443 and 8080 for the web interface. Be sure to not have another service listening on these ports.<br />
It uses a home certificat SSL. <br />
It uses a toml settings file you can modify if you need so.

# TIP

You could create your own global docker-compose service containing for example mysql, redis and maildev as you may use these services in 90% of your project.<br />
And just use their domain in your local project to reach them (or use the network docker feature to reach them directly, your choice)

