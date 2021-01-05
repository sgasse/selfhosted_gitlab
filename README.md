
This repository features a minimal version of a self-hosted Gitlab complete with a CI runner, SSL certificates and an nginx. For small groups/projects, this should be sufficient. For larger projects, at least the CI runners should run on a separate server.

# SSL Setup
To enable communication via SSL, we need valid SSL certificates for our domain. These can be obtained free of charge from the service [LetsEncrypt](https://letsencrypt.org/) with the [Certbot](https://certbot.eff.org/).

To verify that we control the domain which we claim to control, Certbot can request a challenge from the LetsEncrypt servers which will be answered via port 80 in the domain. Thus we need our service to answer on port 80 to obtain the certificates.

In this repository, nginx is setup as it would be once certificates are available. Though for the initial start, we need to change the configuration such that no certificates are required. The exact procedure is described below.

## Launch nginx for initial certificate retrieval
- Comment out the SSL encrypted server block in `main/config/nginx/app.conf`.
- Launch nginx with `docker-compose up --force-recreate -d nginx`
- Obtain certificates for the first time:
```
docker-compose run --rm --entrypoint "\
    certbot certonly --webroot -w /var/www/certbot \
    --email change@me.de \
    -d dev.simongasse.de \
    --rsa-key-size 4096 \
    --agree-tos \
    --force-renewal" certbot
```
- Uncomment the SSL block of nginx and rebuild and restart the container `docker-compose stop nginx && docker-compose up --force-recreate -d nginx`
- Another possibility is to use self-signed certificates, generated with `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout localhost.key -out localhost.crt -config localhost.conf`.

# Launching the setup

A few manual steps are required at the initial startup. They could be automated, but only by introducing unnecessary complexity. Here are the required steps:
- Obtain initial certificates according to the guide in `SSL Setup`.
- Change the gitlab root password in `main/config/gitlab/secrets/gitlab_root_pw.txt`.
- Launch the gitlab instance, certbot and nginx if you did not restart it after obtaining the certificates:
```
docker-compose up -d gitlab-instance
docker-compose up -d certbot
docker-compose up -d nginx
```
- After 2-3mins, try to reach Gitlab at the specified address. You cannot break anything by trying to connect before it is reachable.
- Log into your Gitlab instance, got to the Admin Area (tool symbol on the top bar), navigate to `Shared Runners` and look up the registration token for shared runners.
- Paste the registration token for shared runners in the `docker-compose.yaml` at the service `register-runner`.
- Launch the Docker-in-Docker service, which will separate CI jobs from your host system with `docker-compose up -d docker-in-docker`.
- Register the runner with `docker-compose up register-runner`. This will create a configuration file for the other runner service to use.
- Launch the regular CI runner service with `docker-compose up -d ci-runner`.

