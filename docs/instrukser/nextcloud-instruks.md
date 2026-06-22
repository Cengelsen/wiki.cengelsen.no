---
title: Nextcloud, Snap og Nginx Reverse proxy
date: 2022-09-11
description: En instruks på hvordan du kan kjøre en Nextcloud snap bak en Nginx reverse proxy.
id: 9
---

*NB: Dette er en instruks skrevet i 2022 for min egen hjemmelab den gangen.*

## Hva er Nextcloud? 

Nextcloud er et skylagringsalternativ til Dropbox og Google Drive, som er åpen kildekode. Du kan kjøre det lokalt på din egen maskin, hvilket betyr at du har kontroll over dine filer til enhver tid. Med Nextcloud Hub, satser de på å ha én intregret løsning for fillagring, fildeling, preiketjenester, epost-tjenester og dokumentsamarbeid. I tillegg er det også mulig å installere diverse apper inn i Nextcloud-installasjonen din. Selv om de startet i 2016, så estimerer de selv at det er nå over 400.000 Nextcloud-maskintjenere på nett. 

## Forklaring av problemstilling

I min søken etter svar på internettet, så sitter jeg igjen med inntrykket at denne kombinasjonen av konfigurasjon ikke er vanlig. Derfor skal jeg prøve å være så tydelig som mulig, for å forhindre forvirring og at du kaster bort tiden din.

Det jeg ønsker å forklare i denne instruksen, er hvordan man kan kjøre en Nextcloud instans, installert og konfigurert gjennom Snap, bak en Nginx proxy server. Slik at proxy-serveren tar seg av SSL-sikringen, mens tjeneste-serveren tar seg av alt annet Nginx-relatert.

Om du har konfigurert en Nginx-proxy-server slik som beskrevet i [proxy-instruksen min](nextcloud-snap-og-nginx-reverse-proxy.md), så har du nå to virtuelle maskiner; én virtuell maskin som kjører Nginx som proxy og én virtuell maskin som kjører nginx som en "ordinær" webserver. På den maskinen som kjører en "ordinær" webserver, så skal Nextcloud også kjøre. Problemstillingen er da at Nextcloud må kommunisere riktig med sin lokale "ordinære" webserver, samt at den "ordinære" webserveren må kommunisere riktig med proxy-serveren, slik at jeg kan nå nextcloud-tjenesten ved å gå til *https://eksempel.domene.no*. 

## Forutsetninger

1. Muligheten til å installere Snap.
2. 64bit CPU og 64bit OS er anbefalt.
3. Minst 128MB RAM per prosess, men 512MB RAM per prosess er anbefalt.
4. Nginx er konfigurert som reverse proxy, [slik som beskrevet her](nextcloud-snap-og-nginx-reverse-proxy.md).

## Installere Snap

Installasjon av selve Snap kan variere mellom ulike systemer. En guide for ulike systemer [kan du finne her](https://snapcraft.io/docs/installing-snapd).

## Installere Nextcloud

Det finnes andre måter å installere Nextcloud på som ikke innebærer Snap, men her er poenget at Nextcloud er installert gjennom Snap.

Når du har Snap, kan du følge disse stegene:

1. `sudo snap install nextcloud`
2. `sudo sudo nextcloud.manual-install *sudobruker* *passord*`

## Konfigurere Nextcloud

For at Nextcloud skal kommunisere med Nginx på riktig måte, så trengs det litt konfigurasjon av Nextcloud. Du må gjøre følgende:

1. `sudo snap stop nextcloud`
2. Åpne konfigurasjonsfilen til nextcloud. For Ubuntu 20.04 blir det `/var/snap/nextcloud/31222/nextcloud/config/config.php`.
3. Legg til disse linjene på bunnen av filen:

```
'overwritehost' => 'eksempel.domene.no',
'overwriteprotocol' => 'https',
'overwritewebroot' => '/',
```

samt endre `'trusted_proxies'` til å se slik ut:

```
'trusted_proxies' =>
  array (
    0 => '*Proxy-server IP*',
  ),
```

Om du ikke har `'trusted_proxies'`-variabelen, må du legge den til også.

Det nevnes også mange steder på nettet at du må endre `'overwrite.cli.url'`-variabelen til å være `'https://eksempel.domene.no'`, men jeg har den stående som den er, nemlig `'https://localhost'`.

4. `sudo nextcloud.disable-https`
5. `sudo snap set nextcloud ports.http=*ledig port*`. Det samme er mulig for https: `sudo snap set nextcloud ports.https=*ledig port*`'
6. `sudo snap start nextcloud`

Da har nextcloud all den nødvendige konfigurasjonen for å kunne kommunisere med nginx.

## Konfigurere Nginx

I dette tilfellet så kreves det nødvendig konfigurasjon på både Proxy-serveren og tjeneste-serveren på forhånd for at konfigureringen under skal virke. Da sikter jeg til forbeholdet som denne instruksen tar, hvilket jeg nevnte tidligere.

Nginx-config på Proxy-serveren:

```bash
server {

        server_name *domenenavn*;

        location / {
                include /etc/nginx/proxy_params;

                proxy_pass http://*IP til nextcloud-server*/; # I LXD kan du også bare skrive *container-navn*.lxd

        }


        real_ip_header proxy_protocol;
        set_real_ip_from 127.0.0.1;



    listen [::]:443 ssl; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/*eksempel.domene.no*/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/*eksempel.domene.no*/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}

server {
    if ($host = *eksempel.domene.no*) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

        listen 80 proxy_protocol;
        listen [::]:80 proxy_protocol;

        server_name *eksempel.domene.no*;
    return 404; # managed by Certbot


}
```

Nginx-config på Nextcloud-serveren:

```bash
server {

        listen 80;
        listen [::]:80;

        server_name *eksempel.domene.no*;

        location / {
                proxy_pass_header   Server;
                proxy_set_header    Host $host;
                proxy_set_header    X-Real-IP $remote_addr;
                proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header    X-Forwarded-Proto $scheme;
                proxy_pass          http://127.0.0.1:*nextcloud.http-port*;
        }
}
```

## Kilder

1. [Putting the snap behind a reverse proxy](https://github.com/nextcloud-snap/nextcloud-snap/wiki/Putting-the-snap-behind-a-reverse-proxy), sist lest 11.09.2022.
2. [How To Install and Configure Nextcloud on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-nextcloud-on-ubuntu-20-04), sist lest 11.09.2022.
3. [Nextcloud configuration > Reverse proxy](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/reverse_proxy_configuration.html), sist lest 11.09.2022.
4. [Setting up Nextcloud behind https nginx proxy](https://www.vanwerkhoven.org/blog/2021/setting-up-nextcloud-behind-https-nginx-proxy/), sist lest 11.09.2022.
