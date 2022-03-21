
# Streamline your self-hosting

A guide made initially for my own sake, but I refined it a bit (a lot) so that it can benefit a broader audience.

Shows step by step a possible way to setup and configure a lean mean hosting machine.

### Stack:
- **Traefik** - reverse proxy with dashboard protected by optional basic-auth and/or ip-whitelist.
- **Portainer** - container manager with optional ip-whitelist.
- **Docker Registry** - with optional basic-auth and/or ip-whitelist.
- **Watchtower** - auto update/restart containers.

All services, except Watchtower, is exposed on your sub.domain.whatever with automatic certificate renewal using Traefik/letsencrypt.

With the full stack successfully deployed it is possible to deploy a new container from your own registry configure it to run behind Traefik reverse proxy (benefit from those features) and every time you push a new image to your registry your container service is updated with almost no downtime. With Watchtower configured to do rolling restart a zero-downtime deployment process can be achieved.

This stack is especially handy if combined with something like Github Actions. 

In that particular case Github Actions (or something similar) could be used to run tests and if they turn out successful, push the new image to the registry. This flow could then be configured to run every time a commit to a specific branch is made.

## Prerequisites

- Ubuntu machine.
- Domain names pointed to the machine public IP
`traefik.example.link`, `portainer.example.link` and `registry.example.link` 
## Deployment

To deploy and configure the hosting stack, follow the steps below.

### Install Requirements

#### Step 1 - Update, upgrade and install
Update, upgrade and install a couple of tools that will come in handy.
*(Tool named 'htop' is optional. Pretty cool though)*
```bat
apt-get update && apt-get upgrade -y && apt-get install htop git apache2-utils ufw -y
```

#### Step 2 - Firewall configuration
Configure ufw (ubuntu firewall) to allow http, https and ssh.
```bat
ufw default deny incoming && ufw allow ssh && ufw allow http && ufw allow https && ufw enable && ufw status
```

#### Step 3 - Install Docker and Docker Compose.
Install docker and docker-compose.
```bat
apt-get update && apt-get install ca-certificates curl gnupg lsb-release -y && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && apt-get update && apt-get install docker-ce docker-ce-cli containerd.io -y && apt install docker-compose -y && docker-compose --version
```
### Create files, folders and networks

#### Step 4 - Traefik 
Create the `acme.json` -file to store ssl/tls certificate information.
 ```bat
 mkdir -p /ssl-certs && touch /ssl-certs/acme.json && chmod 600 /ssl-certs/acme.json
 ```

 #### Step 5 - Traefik
 Create the trafik configuration file, `traefik.yml`.
 ```bat
 mkdir -p /etc/traefik/ && touch /etc/traefik/traefik.yml
 ```

 #### Step 6 - Watchtower
 If you want to use a private registry like the one we will setup in this tutorial Watchtower will need a configuration file to be set.
 
 [Link to docs regarding configuration for password protected registry.](https://containrrr.dev/watchtower/private-registries/)

 ```bat
 mkdir -p /etc/watchtower/.docker/ && touch /etc/watchtower/.docker/config.json
 ```

 #### Step 7 - Docker Compose file
 Make the compose folder and file.
 ```bat
 mkdir -p /docker-compose/ci-hosting-stack/ && touch /docker-compose/ci-hosting-stack/docker-compose.yml
 ```

 #### Step 8 - Docker, Volumes and Networks
 Now create some docker volumes and networks we will use in the docker-compose.
 ```bat
 docker network create proxy && docker network create watchtower && docker volume create portainer_data && docker volume create registry_data
 ```
 
### Configure your stack

#### Step 9 - Configure Traefik
Open the Traefik configuration file `traefik.yml`.
```bat
nano /etc/traefik/traefik.yml
```
Paste in the below configuration and update the ``johndoe@your.mail` with your actual mail.
```yml
lobal:
  checkNewVersion: true
  sendAnonymousUsage: false
log:
  level: INFO
api:
  dashboard: true
entryPoints:
  web:
    address: :80
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: :443
certificatesResolvers:
  staging:
    acme:
      email: johndoe@your.mail
      storage: /ssl-certs/acme.json
      caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
      httpChallenge:
        entryPoint: web
  production:
    acme:
      email: johndoe@your.mail
      storage: /ssl-certs/acme.json
      caServer: "https://acme-v02.api.letsencrypt.org/directory"
      httpChallenge:
        entryPoint: web
providers:
  docker:
    exposedByDefault: false
```
*The above configuration has 3 main purposes*
- *Redirect all http to https.*
- *Creates two certificate resolvers. One for testing and another for production.*
- *Exposes the traefik-dashboard.*

When you have updated the mail address you can save and close the file.

To save and close file press: `CTRL+O`, `ENTER` and `CTRL+X`.

#### Step 10 - Setting up the stack `docker-compose.yml`
Open the `docker-compose.yml` -file.
```bat
nano /docker-compose/ci-hosting-stack/docker-compose.yml
```
Paste in the docker-compose code below.

```yml
version: "3"

services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    ports:
      - 80:80
      - 443:443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /etc/traefik:/etc/traefik
      - /ssl-certs:/ssl-certs
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.entrypoints=web,websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=production"
      # Routing.
      - "traefik.http.routers.dashboard.rule=Host(`traefik.example.link`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      # Porting.
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.services.dashboard.loadbalancer.server.port=8080"
      # Authentication.
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$xc2n4OpZ$$.8x7KmQi4w0b4LNYYHHPB0"
      # You may want to whitelist trusted IP's for increased security.
      - "traefik.http.middlewares.dashboard-ipwhitelist.ipwhitelist.sourcerange=[MY.SEC.RET.IP]/32"
      # Middelware chain.
      - "traefik.http.routers.dashboard.middlewares=dashboard-ipwhitelist,dashboard-auth"
      # Exclude from watchtower automatic updates.
      - "com.centurylinklabs.watchtower.enable=false"

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.portainer.tls=true"
      - "traefik.http.routers.portainer.entrypoints=web,websecure"
      - "traefik.http.routers.portainer.tls.certresolver=production"
      # Routing.
      - "traefik.http.routers.portainer.rule=Host(`portainer.example.link`)"
      # Porting.
      - "traefik.http.routers.portainer.service=portainer"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"
      # You may want to whitelist trusted IP's for increased security.
      - "traefik.http.middlewares.portainer-ipwhitelist.ipwhitelist.sourcerange=[MY.SEC.RET.IP]/32"
      # Middelware chain.
      - "traefik.http.routers.portainer.middlewares=portainer-ipwhitelist"
      # Exclude from watchtower automatic updates.
      - "com.centurylinklabs.watchtower.enable=false"

registry:
    image: registry:latest
    container_name: registry
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    volumes:
      - registry_data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.registry.tls=true"
      - "traefik.http.routers.registry.entrypoints=web,websecure"
      - "traefik.http.routers.registry.tls.certresolver=production"
      # Routing.
      - "traefik.http.routers.registry.rule=Host(`registry.example.link`)"
      # Porting.
      - "traefik.http.routers.registry.service=registry"
      - "traefik.http.services.registry.loadbalancer.server.port=5000"
      # Authentication.
      - "traefik.http.middlewares.registry-auth.basicauth.users=admin:$$apr1$$xc2n4OpZ$$.8x7KmQi4w0b4LNYYHHPB0"
      # You may want to whitelist trusted IP's for increased security. Dont do this if you want to use Github Actions without selfhosting runners.
      #- "traefik.http.middlewares.registry-ipwhitelist.ipwhitelist.sourcerange=[MY.SEC.RET.IP]/32"
      # Middelware chain.
      - "traefik.http.routers.registry.middlewares=registry-auth"
      # Exclude from watchtower automatic updates.
      - "com.centurylinklabs.watchtower.enable=false"

  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/watchtower/.docker/config.json:/config.json
    command: --interval 30 --cleanup --label-enable --include-restarting

volumes:
  portainer_data:
    external: true
  registry_data:
    external: true

networks:
  proxy:
    external: true
  watchtower:
    external: true

```

Now before you exit the nano editor you would need to change a few things in the yaml-code.

- Update the basic-auth credentials for Registry and Treafik-dashboard basic-auth middleware.
*You can use the below command to format your username and password to comply with basic-auth format.*
```bat
echo $(htpasswd -nb admin secure_password) | sed -e s/\\$/\\$\\$/g
```
[*Check the Traefik docs on middelware...*](https://doc.traefik.io/traefik/middlewares/http/overview/)

- Update the whitelisted IPs for Portainer and Traefik-dashboard.
*Replace the [MY.SEC.RET.IP] in the yaml-code with your own IP, add more by seperating IPs with commas `127.0.0.0,127.0.0.1`. Using a static VPN IP (selfhosted or provider) when using whitelist options is preferable.*

[*Check the Traefik docs on whitelisting...*](https://doc.traefik.io/traefik/v2.0/middlewares/ipwhitelist/)
 
When all that is out of the way you can save and exit -> `CTRL+O`, `ENTER` and `CTRL+X`.

#### Step 11
Confiure watch tower to work with password protected registry. (like the one that is specifed in the composer)

Open up the `config.json` file in the nano editor.
```bat
nano /etc/watchtower/.docker/config.json
```
Paste the code below.
```json
{
    "auths": {
        "registry.example.link": {
            "auth": "YWRtaW46c2VjdXJlX3Bhc3N3b3Jk"
        }
    }
}
```
The `auth` -string is just your user:pass encoded with base64.
```bat
echo -n 'admin:secure_password' | base64
```

Save and exit the Watchtower config file `CTRL-O` `ENTER` and `CTRL-X`


## Launch compose stack


### Step 12 Run the stack
Run the stack.
```bat
cd /docker-compose/ci-hosting-stack/ && docker-compose up -d && cd ~
```

You can view your running containers with `docker ps` command.

You should now be able to access:
- Traefik Dashboard on `traefik.example.link/dashboard/`
- Portainer on `portainer.example.link`
- Registry on `registry.example.link`

When configuring a new container you should include the labels
1. If you want Watchtower to update the container automaticly.
```
- "com.centurylinklabs.watchtower.enable=true"
```
2. If you want to reverse proxy through Traefik.
```bat
- "traefik.enable=true"
- "traefik.docker.network=proxy"
- "traefik.http.routers.ROUTER_NAME.tls=true"
- "traefik.http.routers.ROUTER_NAME.entrypoints=web,websecure"
- "traefik.http.routers.ROUTER_NAME.tls.certresolver=production"
- "traefik.http.routers.ROUTER_NAME.rule=Host(`[SOME.DOMAIN]`)"
- "traefik.http.routers.ROUTER_NAME.service=ROUTER_NAME"
- "traefik.http.services.ROUTER_NAME.loadbalancer.server.port=[SERVICE.PORT]"
```

I recommend looking through the traefik docs and protainer docs they both have much more to offer.
## Github Actions

If you would like to deploy whatever project you are working on from whatever IDE you, work with you could set up a Github Action to build and push your image whenever you push to a certain branch.

When a new image version is pushed to your registry watchtower will pick it up and update/recreate your deployed container.

Here is an example of how that can be done with Github Actions...
```yml
name: Docker Continues Deployment

on:
  push:
    branches: [ deployment ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Build image
      run: docker build . --file Dockerfile --tag registry.example.link/weather-api:latest
    
    - name: Docker Login
      uses: docker/login-action@v1.14.1
      with:
        registry: registry.example.link
        username: ${{ secrets.REGISTRY_USER }}
        password: ${{ secrets.REGISTRY_PASS }}
    
    - name: Push image
      run: docker push registry.example.link/weather-api:latest
```
