# n8n-newsfeed
# Background
Hello! Welcome to this guide on how to set up an easily customizable and powerful newsfeed which utilizes n8n for automation and Ollama with OpenAI's open weights model (gpt-oss-20b) for AI powered summaries.

This can also be used as a template for different projects.

At least as of right now, this guide will focus on a setup involving both n8n and Ollama running on the same machine (either locally or remotely) as that is what I have done on my server, however, with some modifications to the networking set up you can run the services on different machines.

This project/guide is inspired by [NetworkChuck's](https://www.youtube.com/@NetworkChuck) recent [video on n8n](https://www.youtube.com/watch?v=ONgECvZNI3o) so please make sure to go check out his awesome channel!

Let's get started!

## Installing and running n8n Locally
Below, you will see the steps I took to set it up on my machine along with context why I made the decisions I made, as I followed the documentation, a lot of what is below will follow directly from the n8n or docker documentation.

As I am working in an environment without a GUI, I opted to go for the [docker-compose installation](https://docs.n8n.io/hosting/installation/server-setups/docker-compose/) from the n8n docs instead of using docker-desktop. Please feel free to use Docker Desktop if your prefer.

### Installing Docker
First we must install both Docker Engine and Docker Compose components which will depend on your system. If you choose the Docker Desktop route, it already comes with both of these components by default. 

To install Docker Engine, refer to [this section of the Docker documentation](https://docs.docker.com/engine/install/).

Similarly, to install Docker Compose, refer to [this section of the Docker documentation](https://docs.docker.com/compose/install/).

If using Docker Desktop, refer to [this section of the docker documentation](https://docs.docker.com/desktop/) at the bottom of the page.

To verify that docker is successfully installed, run the following commands:
``` bash
docker --version
docker compose version
```


If all goes well, both commands should output a version.

### (Optional): Non-Root User Access
By default, users have to use `sudo` to be able to use docker. If you would like to permit non-root user access, please refer to [this section of the n8n documentation](https://docs.n8n.io/hosting/installation/server-setups/docker-compose/#2-optional-non-root-user-access).

### DNS Setup
Depending on your networking setup, you may or may not have to set up DNS.

In my case I did not since I opted for adding an entry to the `hosts` file on my main machine to enable remote access to my server.

If you are on Linux, this would be the `/etc/hosts` file, and if you are one Windows, this would be the `C:\Windows\System32\drivers\etc\hosts` file. Just add an entry at the very bottom as follows
```
{remote machine ip}     subdomain.domain
```

If your wish for your setup to be widely and easily accessible by other machines in your network, you should go ahead and setup DNS by referring to [this section of the n8n documentation](https://docs.n8n.io/hosting/installation/server-setups/docker-compose/#3-dns-setup).

### Creating the `.env` File
Create a directory to store your n8n- environment configuration and Docker Compose files, and then navigate to it:
``` bash
mkdir n8n-compose
cd n8n-compose
```

Once inside the `n8n-compose` directory, create a `.env` file to customize your n8n instance's details.
```
# DOMAIN_NAME and SUBDOMAIN together determine where n8n will be reachable from
# The top level domain to serve from
DOMAIN_NAME=example.com

# The subdomain to serve from
SUBDOMAIN=n8n

# The above example serve n8n at: https://n8n.example.com

# Optional timezone to set which gets used by Cron and other scheduling nodes
# New York is the default value if not set
GENERIC_TIMEZONE=Europe/Berlin

# The email address to use for the TLS/SSL certificate creation
SSL_EMAIL=user@example.com
```

Change the above to match your information, but in my case I went with the following:
```
# DOMAIN_NAME and SUBDOMAIN together determine where n8n will be reachable from
# The top level domain to serve from
DOMAIN_NAME=n8nisawesome.local

# The subdomain to serve from
SUBDOMAIN=n8n

# The above example serve n8n at: https://n8n.n8nisawesome.local

# Optional timezone to set which gets used by Cron and other scheduling nodes
# New York is the default value if not set
GENERIC_TIMEZONE=America/Los_Angeles

# The email address to use for the TLS/SSL certificate creation
SSL_EMAIL=<my email>
```

I would suggest keeping the `SUBDOMAIN` as `n8n` and then setting the `DOMAIN_NAME` to whatever domain you would like (for local setup) or the name of the domain that you are using (WAN/internet set up). It is also advisable to have your subdomain + domain match the entry in the `hosts` file.

### Creating Local Files Directory
Now we create a directory called `local-files` for sharing files between the n8n instance running in docker and the host filesystem:
``` bash
mkdir local-files
```

The below Docker Compose file will create this directory, but as indiacted in the docs, doing this manually beforehand avoiod any permission and ownership issues.

### Creating Docker Compose File
Create a file named `compose.yaml` and pase the following in it:
``` yaml
services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.entrypoints=web,websecure
      - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
      - traefik.http.middlewares.n8n.headers.SSLRedirect=true
      - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
      - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
      - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
      - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
      - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
      - traefik.http.middlewares.n8n.headers.STSPreload=true
      - traefik.http.routers.n8n.middlewares=n8n@docker
    environment:
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - N8N_RUNNERS_ENABLED=true
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${GENERIC_TIMEZONE}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files

volumes:
  n8n_data:
  traefik_data:
```

Leave everything as is unless you have a reason to change it.

### Starting Docker Compose
If all went well, we can start n8n by running:
``` bash
sudo docker compose up -d
```

To stop the container, run:
``` bash
sudo docker compose stop
```

### Done
You should now be able to reach n8n using the subdomain + domain combination that you defined in the `.env` configuration.

In my case I do so via `http://n8n.n8nisawesome.local`.

If you browser tells you that danger may lie ahead, you can safely ignore this as we are running this ourselves so we know that its not malicious. The browser does this most likely due to self signed TSL/SSL certificates.

As recommended in the docs, if you are having trouble reaching your instance, check your server's firewall settings and your DNS configuration.

## Installing, Setting up, and Running Ollama Locally
### Installing Ollama
For general instruction refer to the [instructions on the Ollama repository](https://github.com/ollama/ollama).

As I am on Linux, I used the following command to install Ollama:
``` bash
curl -fsSL https://ollama.com/install.sh | sh
```

You can also opt to run Ollama via docker, but I won't cover that here.

### Pullilng the Model
Once Ollama is installed we can now pull the desired model. In my case I opted to go for the `gpt-oss:20b` model which can be installed with the following command:
``` bash
ollama pull gpt-oss:20b
```

This may take a second (or a while longer) as the model is around 13Gb at the time of writing.

### Networking
In order to have Ollama be accessible to the Docker container running n8n, we need to do a little bit of networking.

First we make Ollama listen on all network interfaces rather than just localhost. We can do so as follows on Linux:
``` bash
export OLLAMA_HOST=0.0.0.0:11434
```

The above will only make the change temporarily, if we want this to be permanent we have to edit the systemd service as below.

First run the command `sudo systemctl edit ollama.service`, then add the following line:
``` ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Finally run the following to reexecute systemd:
```
sudo systemctl daemon-reexec
```

If necessary (Ollama is currently running), also make sure to restart ollama:
```
sudo systemctl restart ollama.service
```

Next, we just need to open the firewall for Ollama's port with the following commands:
``` bash
sudo firewall-cmd --add-port=11434/tcp --permanent
sudo firewall-cmd --reload
```

Tip: if you are having some trouble with this step, you can use the below approach for debugging.

To test if the container can reach Ollama, we can drop into a shell in our n8n container with `sudo docker exec -it --user root n8n-compose-n8n-1 sh` (if this doesn't work for your container, search up which command you need to use to get into a shell on your container) and use `curl` to check if we get anything from Ollama with `curl http://192.168.12.134:11434/api/tags`.

If curl is not installed in your container you can do so with `apk add --no-cache curl` (again this works for my alpine linux container, if yours is different, you may need to do some searching).

If all goes well, the command `curl http://192.168.12.134:11434/api/tags` should output some JSON.

### Running Ollama
Now all that we have everthing set up, all that remains is to run the ollama service.

We can do so either via the command:
``` bash
ollama serve
```

or

``` bash
sudo systemctl start ollama.service
```

It just depends if you would rather run it manually or use the systemd module.

To stop ollama via the systemd module, you can use:
``` bash
sudo systemctl stop ollama.service
```

## Configuring our Workflow in n8n
Once both n8n and Ollama, with our `gpt-oss:20b` model, are running, we can move onto the fun part, which is actually creating our workflow on n8n!