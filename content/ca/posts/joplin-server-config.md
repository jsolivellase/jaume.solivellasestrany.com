+++
date = '2025-08-06'
tags = ['joplin', 'notes', 'self-hosting']
title = 'Configuració de Joplin Server'
+++

Des de fa un temps, utilitz *Joplin* per prendre notes en format digital. Vaig migrar des de *Notion*, ja que cercava una aplicació senzilla i ràpida per crear notes usant *Markdown*, *Notion* fa massa coses i no és gaire ràpida.

La veritat és que estic molt content, és molt fàcil d'emprar, disposa d'aplicació per a *Android*, permet sincronitzar entre dispositius fent servir *Dropbox*, *OneDrive*, *Nextcloud*, etc. A més a més, pels usuaris que necessitin funcionalitats més avançades, hi ha un conjunt de complements creats per la comunitat per afegir pràcticament qualsevol funcionalitat que et faci falta, enllaços entre notes, plantilles, taules de continguts, etc.

Recentment, he tingut la necessitat de compartir notes amb la meva dona i, per aconseguir-ho, he hagut de configurar la sincronització amb *Joplin Server*. L'opció de compartir notes entre diferents usuaris només està disponible si fas ús de *Joplin Cloud* o si et configures el teu propi servidor.

[La documentació](https://github.com/laurent22/joplin/blob/dev/packages/server/README.md) és pobra en alguns apartats i he tingut alguns problemes per configurar alguns detalls, anem a veure com ho he configurat.

## Configuració bàsica del servidor

Com assenyala la documentació, utilitzarem *Docker Compose* per executar el servidor.

1. Crea un nou fitxer de *Compose* on desitgis amb el següent contingut (pots trobar la darrera versió [aquí](https://raw.githubusercontent.com/laurent22/joplin/dev/docker-compose.server.yml)):
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
2. A la mateixa ubicació, crea un document amb el nom `.env` ([darrera versió](https://raw.githubusercontent.com/laurent22/joplin/dev/.env-sample)) que contindrà les variables d'entorn necessàries per executar correctament el servidor:
```env
POSTGRES_USER=admin
POSTGRES_PASSWORD='supersecretpassword'
POSTGRES_DATABASE=joplin
POSTGRES_PORT=5432
APP_BASE_URL=http://localhost:22300
```
3. Inicialitza el servidor amb la instrucció `docker compose up -d`.

Amb aquests senzills passos, ja tindrem el servidor aixecat amb la seva base de dades on es guardaran totes les nostres notes.

Per ventura, tenir tota la informació guardada dins la base de dades, és útil pel teu cas d'ús, però personalment, preferesc que les notes es guardin en el disc dur o a un *storage* d'AWS per simplificar el procés de realitzar còpies de seguretat.

## Utilitzar el sistema de fitxers local per a guardar-hi les notes

En el meu cas, el servidor de *Joplin* s'executa a una *Raspberry Pi* que tenc a casa i, aquesta, ja disposa d'un sistema per fer còpies de seguretat dels fitxers del disc dur. Per tant, he configurat el servidor per guardar les notes en el disc dur en lloc d'un emmagatzematge al núvol.

Per aconseguir-ho, hem de configurar la variable d'entorn `STORAGE_DRIVER` de la següent forma:
```docker
STORAGE_DRIVER=Type=Filesystem; Path=/path/to/dir
```

A més a més, com he dit, vull disposar dels fitxers en el meu disc dur local, aleshores, és necessari muntar un volum per poder-hi accedir des de l'amfitrió. El fitxer de *Compose* quedaria de la següent forma:
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

Aquí apareix el primer problema que he tengut. Si tornem a engegar el servidor amb les instruccions `docker compose down` i `docker compose up -d`, veurem en els registres que el servidor **no aconsegueix arrancar**. La causa és que la imatge del servidor fa ús d'un usuari sense permisos elevats, per tant, no té accés al volum de la màquina de l'amfitrió. Això és una bona pràctica, ja que utilitzar usuaris amb permisos elevats pot comprometre la màquina amfitriona. Normalment, moltes imatges de *Docker* disposen d'algun argument per indicar quin usuari vols fer anar per executar la imatge, d'aquesta manera, pots donar-li els permisos necessaris per accedir al que faci falta. Desgraciadament, no és el cas d'aquesta.

Per solucionar-ho, el que he hagut de fer és donar-li permisos a l'usuari i al grup creats per la imatge a la carpeta del volum amb la instrucció `sudo chown 1001:1001 /local/path/to/dir`. L'usuari que utilitza la imatge és `joplin` del grup `joplin` ambdós amb l'identificador `1001`.

Fet això, si reiniciem el servidor veurem que aquesta vegada sí que aconsegueix engegar, ja que ja disposa dels permisos necessaris per escriure al volum que hem muntat.

Ara que ja tenim el servidor funcionant i guardant les notes en el nostre sistema de fitxers, és l'hora de configurar l'accés des de fora de la nostra xarxa, perquè volem poder accedir i sincronitzar les nostres notes fora de casa.

## Configuració del reverse proxy

Hi ha moltes maneres d'accedir a la nostra xarxa local des de l'exterior, per exemple podríem configurar una VPN fent ús de *Tailscale*. En el meu cas, tenc uns quants dominis en propietat i, el més senzill i pràctic, és configurar un *reverse proxy* per poder accedir al servidor mitjançant una adreça web. D'aquesta manera no és necessari connectar-me a una VPN per sincronitzar les dades.

També tenim diverses opcions per muntar un *reverse proxy*, per exemple utilitzant un servidor HTTP com *NGINX* o *Apache*. Una vegada més, per simplicitat, preferesc fer servir *Cloudflare Tunnel*.

*Cloudflare Tunnel* és una eina que ens permet crear una connexió entre el nostre servidor i els de *Cloudflare* sense necessitat de tenir una IP pública estàtica. Això fa que si tenim una IP pública que va canviant o estem darrere un [CGNAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT), la connexió al nostre servidor sempre estirà assegurada gràcies al túnel creat. A més a més, *Cloudflare* ens ofereix una sèrie d'eines de seguretat per prevenir atacs i, tot això, gratuïtament.

Aquesta eina ja l'he usada per configurar el *Nextcloud* on anteriorment sincronitzava les notes, en [aquest enllaç](https://github.com/jsolivellase/nextcloud-aio-cloudflare-tunnel) tenc l'explicació de com configurar el túnel. L'adreça que hem d'utilitzar en el túnel és `http://app:22300`. Usem `app` en lloc de `localhost` perquè a la mateixa xarxa tenim tres serveis, la base de dades (`db`), el túnel (`cloudflare-tunnel`) i el servidor (`app`).

El fitxer de *Compose* amb el túnel configurat quedaria de la següent manera:
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

També hem de modificar el fitxer `.env` per afegir-hi el *token* del túnel i actualitzar la variable `APP_BASE_URL` amb l'adreça que hem configurat en el túnel:
```env
CLOUDFLARE_TUNNEL_TOKEN='supersecrettoken'
POSTGRES_USER=admin
POSTGRES_PASSWORD='supersecretpassword'
POSTGRES_DATABASE=joplin
POSTGRES_PORT=5432
APP_BASE_URL=https://joplin.example.com
```

Fet això, si reiniciem el servidor i accedim a l'adreça web que acabam de configurar `https://joplin.example.com`, hauriem de veure la pantalla d'autenticació. Per defecte, l'usuari és **admin@localhost** i la contrasenya **admin**. Una vegada autenticats, hauríem de canviar les credencials per motius de seguretat.

Fer el canvi de contrasenya no és un problema, però si volem canviar el correu d'usuari o crear-ne un de nou, s'ha d'enviar un correu i, en cap moment hem configurat un servidor de correu per poder enviar-los. Aleshores, aquí sorgeix el darrer problema a arreglar abans de tenir el servidor totalment configurat.

## Configuració del servidor de correu

La configuració del servidor de correu és bastant senzilla, tot i que no està documentada. Només fa falta afegir unes variables d'entorn al nostre fitxer de *Compose* i quedarà configurat.

Personalment, no dispòs de cap servidor de correu, per tant, he utilitzat un compte de *Gmail*, pel fet que ofereix un servei per enviar correus fent ús del teu compte. Pots seguir les instruccions [d'aquest article](https://support.google.com/a/answer/176600?hl=en) per configurar-ho, jo he usat la segona opció, usant una contrasenya d'aplicació.

Una vegada hem recopilat tota la informació del servidor SMTP, afegirem les variables d'entorn necessàries. El fitxer de *Compose* quedaria de la següent forma:
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
No oblidis afegir la contrasenya al fitxer `.env`:
```env
CLOUDFLARE_TUNNEL_TOKEN='supersecrettoken'
POSTGRES_USER=admin
POSTGRES_PASSWORD='supersecretpassword'
POSTGRES_DATABASE=joplin
POSTGRES_PORT=5432
APP_BASE_URL=https://joplin.example.com
MAILER_AUTH_PASSWORD=supersecretapppassword
```

Finalment, tenim el servidor totalment configurat. Per comprovar que tot funciona, podem crear un nou usuari o canviar el correu de l'administrador. En els dos casos, hauríem de rebre un correu amb la informació.

## Migració de les notes

Si com en el meu cas, ja estàveu sincronitzant les notes a una altra banda, la migració és molt senzilla. Simplement, sincronitza per darrera vegada les notes per tenir la darrera versió d'aquestes, una vegada fet això, canvia la configuració de sincronització i apunta al nou servidor. Automàticament, sincronitzarà totes les notes que tens en local al nou servidor. Pensa a fer aquests canvis a tots els dispositius on utilitzis *Joplin*.

---

Ara si, ja tenim el nostre servidor configurat i podem crear nous comptes i compartir les nostres notes amb qui vulguem. Ara falta la part més important, continuar escrivint i compartint 😉.
