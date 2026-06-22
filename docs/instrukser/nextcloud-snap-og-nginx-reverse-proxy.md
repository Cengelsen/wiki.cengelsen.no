---
title: Nginx som reversert proxy i LXD
date: 2022-09-11
description: En instruks på hvordan man kan rigge opp Nginx som en proxy-server i LXD.
id: 6
---

*NB: Dette er en instruks skrevet i 2022 for min egen hjemmelab den gangen.*

## Problemstilling

Det jeg ønsker å gjøre er å ha en felles container som håndterer all "reversert proxy"-omdirigering og SSL-terminering for alle de andre containerne på serveren. Jeg ønsker å bruke Nginx for dette formålet. 

## Forklaring

Det kan kanskje være litt vanskelig å se for seg hvordan logistikken av det blir. Det du forhåpentligvis sitter igjen med etter denne instruksen er én LXD-container som fungerer som en Nginx proxy-server, og minst én LXD-container som holder på tjenesten du ønsker å eksponere til internett. Poenget er da at proxy-serveren tar imot alle forespørsler fra internett og viderefører trafikken til containeren som holder på tjenesten. Både proxy-containeren og tjeneste-containeren kjører begge en instans av Nginx som "kommuniserer" med hverandre for å dirigere nett-trafikken korrekt.

## Installere og konfigurere LXD

Denne instruksen har som forutsetning at du har installert og konfigurert LXD på serveren din. Du kan [følge instruksen min](lokal-lxd.md) for å gjøre det.

Når dette er gjort, så må du oppprette "enheter" for proxy-containeren. Utenfor proxy-containeren, må du kjøre:

```bash
lxc config device add *navn_på_container* *navn_på_enhet* proxy listen=tcp:0.0.0.0:80 connect=tcp:127.0.0.1:80 proxy_protocol=true

lxc config device add *navn_på_container* *navn_på_enhet* proxy listen=tcp:0.0.0.0:443 connect=tcp:127.0.0.1:443 proxy_protocol=true
```

Dette er hva de ulike parameterne betyr:

| Parameter | Forklaring |
| --------- | ---------- |
| *navn_på_container* | Navnet på proxy-containeren |
| *navn_på_enhet* | Navnet for "enheten" du oppretter |
| proxy | Hvilken type enhet du oppretter |
| listen= tcp:0.0.0.0:80 | Proxy-enheten skal lytte på verten på port 80, protokoll TCP, på alle grensesnitt |
| connect= tcp:127.0.0.1:80 &nbsp; &nbsp; | Proxy-enheten skal koble seg til containeren på port 80, protokoll TCP, på tilbakefôringsgrensesnittet. Det er ikke mulig å skrive "localhost", bare IP-adressen, i LXD-versjoner >= 3.13. |
| proxy_protocol | Ber om å aktivere proxy-protokollen, slik at den reverserte proxyen får den opprinnelige IP-adressen fra proxy-enheten |

Om du vil fjerne proxy-enheten, kan du skrive: 

`lxc config device remove *navn_på_container* *navn_på_enhet*`

## Installere Nginx

Hvordan du installerer Nginx varierer ut ifra hvilket system du bruker. [Her er en instruks for hvordan å installere Nginx på ulike systemer](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/). 

## Konfigurere Nginx i tjeneste-containeren

Det trengs litt konfigurering av Nginx-en som kjører i tjeneste-containeren.

Opprett `/etc/nginx/conf.d/real-ip.conf` i tjeneste-containeren:

```sh
real_ip_header    X-Real-IP;
set_real_ip_from  *navn_på_proxy_container*.lxd;
```	

Opprett en Nginx-konfig, `/etc/nginx/sites-available/*konfig-navn*`, i tjeneste-containeren:

```sh
server {
        listen 80;
        listen [::]:80;

        server_name *domenenavn*;

        root /path/to/website/folder;
        index index.html;

        location / {try_files $uri $uri/ =404;
        }	
}
```

Denne konfigurasjonsfilen kan variere avhengig av kravene tjenesten stiller til Nginx-konfigurasjonen. 

Eksemplet over er for å servere en statisk nettside. Her trengs ikke SSL-terminering, siden proxy-serveren håndterer det.

## Konfigurere Nginx i proxy-containeren

Opprett en Nginx-konfig, `/etc/nginx/sites-available/*konfig-navn*`, i proxy-containeren:

```sh
server {
        listen 80 proxy_protocol;
        listen [::]:80 proxy_protocol;

        server_name *domenenavn*;

        location / {
                include /etc/nginx/proxy_params;

                proxy_pass http://*navn_på_tjeneste_container*.lxd;
        }

        real_ip_header proxy_protocol;
        set_real_ip_from 127.0.0.1;
}
```

Skaff SSL-sikring gjennom Certbot. Dette er en fremgangsmåte i Ubuntu 20.04:

1. `lxc shell *proxy_container_navn*`
2. `sudo add-apt-repository ppa:certbot/certbot`
3. `sudo apt-get install certbot python-certbot-nginx`
4. `sudo certbot --nginx`
  - Agree
  - No
  - *velg riktig domene*
  - 2 (Redirect)
5. Endre de nye linjene i nginx-konfigen til å se slik ut:

```
listen 443 ssl proxy_protocol; # managed by Certbot
listen [::]:443 ssl proxy_protocol; # managed by Certbot
```

6. `sudo systemctl restart nginx`

## Kilder

1. [A Beginner's Guide to LXD: Setting Up a Reverse Proxy to Host Multiple Websites](https://www.linode.com/docs/guides/beginners-guide-to-lxd-reverse-proxy/), sist lest 12.09.2022.
2. [Nginx > Tutorials > Install](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/), sist lest 12.09.2022.
