+++
date = '2025-08-06'
tags = ['joplin', 'notes', 'self-hosting']
title = 'Joplin Server configuration'
+++

For a while now, I have been using Joplin as a note-taking app. I migrated from Notion because I was looking for a simple note-taking app compatible with Markdown. Notion is too complex; it does a lot of things, and it's not the fastest.

I am very happy with the change, it's straightforward to use, it has an application for Android, and it has synchronization across devices using your Dropbox, OneDrive, Nextcloud or its cloud (Joplin Cloud). Moreover, for the advanced users, it has a lot of plugins to add any feature you could possibly need, like linked notes, templates, table of contents, etc.

Recently, I have needed to share notes with my wife. To be able to do it, I have had to configure Joplin Server. The option to share notes between users is only available for users of Joplin Cloud or users who self-host Joplin Server.

[The documentation](https://github.com/laurent22/joplin/blob/dev/packages/server/README.md) is a bit short on some sections, and I have had some issues configuring some parts. Let's see how I have solved it.

## Base configuration of the server

As we can see in the docs, we will use Docker Compose to launch the server.

1. Create a Compose file where you want with the following contents (you can find the last sample version [here](https://raw.githubusercontent.com/laurent22/joplin/dev/docker-compose.server.yml)):
```docker
version: '3'

services:
    db:
        image: postgres:16
        volumes:
            - ./data/postgres:/var/lib/postgresql/data
        ports:
            - "5432:5432"
        restart: unless-stopped
        environment:
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_DB=${POSTGRES_DATABASE}
    app:
        image: joplin/server:latest
        depends_on:
            - db
        ports:
            - "22300:22300"
        restart: unless-stopped
        environment:
            - APP_PORT=22300
            - APP_BASE_URL=${APP_BASE_URL}
            - DB_CLIENT=pg
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_DATABASE=${POSTGRES_DATABASE}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_PORT=${POSTGRES_PORT}
            - POSTGRES_HOST=db

```
2. In the same path, create an `.env` file ([last sample version](https://raw.githubusercontent.com/laurent22/joplin/dev/.env-sample)) that will contain the required environment variables to run the server:
```env
POSTGRES_USER=admin
POSTGRES_PASSWORD='supersecretpassword'
POSTGRES_DATABASE=joplin
POSTGRES_PORT=5432
APP_BASE_URL=http://localhost:22300
```
3. Run the server with `docker compose up -d`.

After these steps, we will have the server up and running with its database where notes will be stored.

Maybe, having all the data on the database is what you want for your use case, but in my case, I prefer to store my notes on the local file system or in some cloud storage like AWS. This makes the process of backing up data easier.

## Using the local file system to store notes

In my case, Joplin server is running on a Raspberry Pi at home. This one already has a backup system in place for any file on the hard disk. That's the reason I have configured the server to save the notes to the hard disk instead of the cloud.

To achieve it, we need to add the environment variable `STORAGE_DRIVER` like you can see in the following example:
```docker
STORAGE_DRIVER=Type=Filesystem; Path=/path/to/dir
```

Moreover, as I said before, I want to be able to access to the files from my local file system then, it's mandatory to mount a volume to get access from the host. Here is the updated Compose file:
```docker
version: '3'

services:
    db:
        image: postgres:16
        volumes:
            - ./data/postgres:/var/lib/postgresql/data
        ports:
            - "5432:5432"
        restart: unless-stopped
        environment:
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_DB=${POSTGRES_DATABASE}
    app:
        image: joplin/server:latest
        volumes:
            - /local/path/to/dir:/path/to/dir:rw
        depends_on:
            - db
        ports:
            - "22300:22300"
        restart: unless-stopped
        environment:
            - APP_PORT=22300
            - APP_BASE_URL=${APP_BASE_URL}
            - DB_CLIENT=pg
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_DATABASE=${POSTGRES_DATABASE}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_PORT=${POSTGRES_PORT}
            - POSTGRES_HOST=db
            - STORAGE_DRIVER=Type=Filesystem; Path=/path/to/dir

```

Here comes the first issue I have faced. If we try to run the server executing `docker compose down` and `docker compose up -d`, we will see in the logs that the **server is unable to boot up**. Joplin server image is using a non-root user, then the user doesn't have access to the host folder. Using non-root user is a good practice to prevent containers from accessing to the host resources. Sometimes, Docker images have an argument to specify which user to use inside the container so, you can grant any required permission to get access to what is needed. Sadly, it's not the case with this image.

In order to solve it, I have granted permissions on the folder to the container user using the UUID and GUID with the instruction `sudo chown 1001:1001 /local/path/to/dir`. The user used on the container is `joplin` from group `joplin` both of them with the UID `1001`.

After doing these steps, if we restart the server, it will boot up without any problem because the user has the proper permissions to write to the volume.

The server is up and running and storing notes on our hard disk; it's time to configure the access to it from outside our home network because, in the end, we want to synchronize our notes in any place.

## Configure the reverse proxy

There are a lot of ways to get access to our home network from the outside. We can configure a VPN using Tailscale, for example. In my situation, I have some domains on property, and it's easier and more useful to configure a reverse proxy to get access to the server from a web address. With this solution, we get rid of the need to use a VPN.

There are some options regarding the reverse proxy too. We can configure an HTTP server like NGINX or Apache. One more time, for simplicity’s sake, I prefer to use Cloudflare Tunnel.

Cloudflare Tunnel is a tool that provides you with a secure way to connect your resources to Cloudflare without a publicly routable IP address. It works too if we are behind a [CGNAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT). Moreover, it offers some tools to prevent us from malicious attacks. All this for free.

I have used this tool before to configure Nextcloud which I was using to synchronize my notes. In [this link](https://github.com/jsolivellase/nextcloud-aio-cloudflare-tunnel), you can find the steps to configure the tunnel. The path to configure for the tunnel is `http://app:22300`. We are using `app` instead of `localhost` because we have three services inside the network: the database (`db`), the tunnel (`cloudflare-tunnel`) and the server (`app`).

The updated Compose file should look like this:
```docker
version: '3'

services:
    db:
        image: postgres:16
        volumes:
            - ./data/postgres:/var/lib/postgresql/data
        ports:
            - "5432:5432"
        restart: unless-stopped
        environment:
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_DB=${POSTGRES_DATABASE}
    app:
        image: joplin/server:latest
        volumes:
            - /local/path/to/dir:/path/to/dir:rw
        depends_on:
            - db
        ports:
            - "22300:22300"
        restart: unless-stopped
        environment:
            - APP_PORT=22300
            - APP_BASE_URL=${APP_BASE_URL}
            - DB_CLIENT=pg
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_DATABASE=${POSTGRES_DATABASE}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_PORT=${POSTGRES_PORT}
            - POSTGRES_HOST=db
            - STORAGE_DRIVER=Type=Filesystem; Path=/path/to/dir
    cloudflare-tunnel:
        image: cloudflare/cloudflared:latest
        restart: unless-stopped
        command: tunnel --no-autoupdate run
        environment:
        - "TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}"

```

We need to modify the `.env` file too to add the *token* of the tunnel and update the environment variable `APP_BASE_URL` with the address we have configured in Cloudflare:
```env
CLOUDFLARE_TUNNEL_TOKEN='supersecrettoken'
POSTGRES_USER=admin
POSTGRES_PASSWORD='supersecretpassword'
POSTGRES_DATABASE=joplin
POSTGRES_PORT=5432
APP_BASE_URL=https://joplin.example.com
```

Again, if we restart the server and go to the configured address `https://joplin.example.com`, we should see the login page. By default, the credentials are **admin@localhost** for the user and **admin** for the password. Once we are in, we must change the credentials for security reasons.

Changing the password is not a problem, but if we want to change the admin email or create a new user, it's mandatory to send an email, and we didn't configure any mail server to be able to send them. So, here is the last stone in our path before having the server fully configured.

## Configure the mail server

Mail server configuration is pretty easy, but it's not documented. It's necessary to add some environment variables to our Compose file, and done.

I don't have any configured mail server, so I have used a Gmail account because it offers a service to send emails using it. You can follow the instructions [from this article](https://support.google.com/a/answer/176600?hl=en) to set it up. I have used the second option, app password.

Once we have everything in place, and we have gathered the SMTP server information, it's time to add the required environment variables. Here we have our updated Compose file:
```docker
version: '3'

services:
    db:
        image: postgres:16
        volumes:
            - ./data/postgres:/var/lib/postgresql/data
        ports:
            - "5432:5432"
        restart: unless-stopped
        environment:
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_DB=${POSTGRES_DATABASE}
    app:
        image: joplin/server:latest
        volumes:
            - /local/path/to/dir:/path/to/dir:rw
        depends_on:
            - db
        ports:
            - "22300:22300"
        restart: unless-stopped
        environment:
            - APP_PORT=22300
            - APP_BASE_URL=${APP_BASE_URL}
            - DB_CLIENT=pg
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_DATABASE=${POSTGRES_DATABASE}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_PORT=${POSTGRES_PORT}
            - POSTGRES_HOST=db
            - STORAGE_DRIVER=Type=Filesystem; Path=/path/to/dir
            - MAILER_ENABLED=1
            - MAILER_HOST=smtp.gmail.com
            - MAILER_PORT=587
            - MAILER_SECURITY=starttls
            - MAILER_AUTH_USER=youremail@gmail.com
            - MAILER_AUTH_PASSWORD=${MAILER_AUTH_PASSWORD}
            - MAILER_NOREPLY_NAME=JoplinServer
            - MAILER_NOREPLY_EMAIL=youremail@gmail.com
    cloudflare-tunnel:
        image: cloudflare/cloudflared:latest
        restart: unless-stopped
        command: tunnel --no-autoupdate run
        environment:
        - "TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}"

```
Keep in mind to add the app password to the `.env` file:
```env
CLOUDFLARE_TUNNEL_TOKEN='supersecrettoken'
POSTGRES_USER=admin
POSTGRES_PASSWORD='supersecretpassword'
POSTGRES_DATABASE=joplin
POSTGRES_PORT=5432
APP_BASE_URL=https://joplin.example.com
MAILER_AUTH_PASSWORD=supersecretapppassword
```

At the end, we have our server configured. To check everything is working, we can add a new user or change the admin email. In both cases, we must receive an email with the information.

## Migrate the notes

If you already have configured some synchronization tool like me, the migration between tools is straightforward. First, synchronize the notes in all devices for the last time to have the latest version. After synchronization is done, change the synchronization configuration to the configured server. Automatically, the app will synchronize all the local notes to the server. Keep in mind to do these steps on all of your devices where you are using Joplin.

---

That's it, we have our server up and running; we can add new accounts and share notes with other users. Now, it's time for the most important part: keep writing and sharing 😉.