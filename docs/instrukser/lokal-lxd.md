---
title: Hvordan rigge opp LXD lokalt
date: 2022-07-24
description: En instruks på hvordan rigge opp LXD lokalt for virtualisering.
id: 5
---

*NB: Dette er en instruks skrevet i 2022 for min egen hjemmelab den gangen*

## Hva er LXD?

LXD er en CLI-tjeneste for å forvalte og drifte virtuelle maskiner og beholdere. LXD er skrevet i Go og er lisensiert under Apache2-lisensen. LXD kan deles opp i LXD (Linux Daemon) og LXC (Linux Containers). 

LXC er programvaren som muliggjør skapelsen av virtuelle maskiner og beholdere. LXD er bygget på toppen av LXC, og visstnok forbedrer LXC med bedre sikkerhet, skalerbarhet, brukervennlighet og prosesseringskostnad. En annen liten forskjell er at LXC bruker C API'et, mens LXD bruker REST API'et.

LXD kan også [integreres inn i andre plattformer og verktøy](https://linuxcontainers.org/lxd/third-party-integrations/), som feks. Ansible, Terraform og Juju. 

Før du installerer det lokalt, kan du også [prøve en demo av det her](https://linuxcontainers.org/lxd/try-it/)!

## Formålet med denne instruksen

Denne instruksjonen har som formål å veilede deg til å installerere og konfigurere LXD, slik at du kan bruke LXD lokalt for enkel virtualisering. Etter denne instruksjonen burde du sitte igjen med et LXD-miljø hvor du kan skape nye beholdere, starte og stoppe dem og slette dem, samt at disse beholderne kan kommunisere med hverandre. Det er selvfølgelig mer avanserte ting en kan gjøre med LXD, men det dekkes ikke her. 

## Avhengigheter

Om du installerer det gjennom Snap, er det anbefalt at man har minst 2 GB RAM. Utenom det er det også anbefalt at det brukes et ZFS-filsystem som "storage pool" for LXD. I så tilfelle, krever LXD at du har installert 
```zfsutils-linux```.

Man er også avhengig av at moderkortet og prosessoren støtter virtualisering.

## Installasjon

### Snap

LXD er tilgjengelig for nedlasting og installering gjennom Snap. Om du har Snap, kan du kjøre ```snap install lxd```. Siden Ubuntu 20.04, er LXD allerede installert, som en Snap-pakke, etter en fersk installasjon av Ubuntu.

Installasjon av selve Snap kan variere mellom ulike systemer. En guide for ulike systemer [kan du finne her](https://www.ubuntupit.com/how-to-install-snap-package-manager-in-linux-distributions/).

### Manuelt

Siden LXD kommer ferdig installert med Ubuntu, trengte jeg bare å tenke på konfigurering. 

På mange operativsystemer kommer ikke Snap med en fersk installasjon, men man må heller ikke bruke Snap for å installere det. [Her er en oversikt](https://linuxcontainers.org/lxd/getting-started-cli/) for installasjon på et par ulike systemer uten Snap.

## Konfigurasjon

### Bruker-rettigheter

Legg til brukeren din til lxd-gruppen.:

```sudo adduser <navn på bruker> lxd```

Du kan bekrefte ved å skrive ```id -nG```. Om lxd står i den listen, har brukeren lxd-rettigheter.

### Lagringsplass

Før du initialiserer LXD, er det viktig at du har satt tilside lagringsplassen du vil bruke. LXD skaper en ZFS-pool på den dedikerte lagringsplassen du har satt tilside, så du trenger ikke tenke på ZFS-konfigurering på forhånd. 

### Initialisering

Om det er første gangen LXD kjører på maskinen, så må en først initialisere med ```lxd init```. I denne prosessen får man en serie med spørsmål for å konfigurere lagringsplassen som LXD skal bruke. De er som følger:

```sh
Do you want to configure a new storage pool? (yes/no) [default=yes]: yes
Name of the new storage pool [default=default]: <velg navn>
Name of the storage backend to use (btrfs, dir, lvm, zfs) [default=zfs]: zfs
Create a new ZFS pool? (yes/no) [default=yes]: yes
Would you like to use an existing block device? (yes/no) [default=no]: yes
Path to the existing block device: path/to/storage/device/<name of storage device>
```

Etter dette er konfigurering av lagringsplass ferdig.

### Nettverk

I samme initialisering blir også nettverket til LXD konfigurert. I samme stil som i sted, får du en serie med spørsmål. De er som følger:

```sh
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to create a new local network bridge? (yes/no) [default=yes]: yes
What should the new bridge be called? [default=lxdbr0]: lxdbr0
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: auto
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: auto
```

Dette åpner for følgende egenskaper:

- Hver beholder blir automatisk tildelt en privat IP-adresse.
- Hver beholder kan kommunisere med hverandre over det private nettverket.
- Hver beholder kan initiere kontakt med internett
- Hver beholder forblir utilgjengelig fra internett. En kontakt kan IKKE initieres fra internett for å nå beholderen.

### Diverse

Du får også 3 spørsmål om diverse ting:

```sh
Would you like LXD to be available over the network? (yes/no) [default=no]: no
Would you like stale cached images to be updated automatically? (yes/no) [default=yes] yes
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: no
```

Etter dette, kjører et script i bakgrunnen. Om du ikke får noe utfôr, så er det helt normalt.

## Håndtere beholdere

Selv om tjenesten heter LXD spå bruker man kommandoen ```lxc``` for å kommunisere med LXD Hypervisor'en

### Se oversikt

For å liste opp alle beholderne du har, skriv ```lxc list```. For å se liste over tilgjengelige kommandoer, skriv
```lxc -h```.

### Skape en beholder

For å skape en ny beholder, skriv ```lxc launch <navn på OS>:<versjon> <navn på beholderen>```. Så om du vil skape en beholder med ubuntu 20.04 og med navn *webserver*, skriver du ```lxc launch ubuntu:20.04 webserver```. Dette kommer til å skape og starte beholderen.

Teknisk sett er <navn på OS> identifikatoren for den forhånds-konfigurerte listen av LXD-avbildningsfiler. <versjon> er navnet på avbildningsfilen. 20.04 er en forkortelse for Ubuntu 20.04. 

For å se en komplett liste over tilgjengelige avbildningsfiler, skriv ```lxc image list images:```. Du kan også begrense listen til bare Ubuntu, ved å skrive ```lxc image list ubuntu:```. Du kan hente ut info om avbildningsfilen ved å skrive ```lxc image info <navn på OS>:<versjon>```. 

### Start/stopp/slett en beholder

Starte beholderen: ```lxc start <navn på beholderen>```. 

Stoppe beholderen: ```lxc stop <navn på beholderen>```.

Slette beholderen: ```lxc delete <navn på beholderen>```.

### Gi en beholder statisk IP-adresse

LXD har en innebygget DHCP-server og kan tildele en tilfeldig IP-adresse til alle beholderne. Den initielle IP-adressen vedvarer selv om beholderen omstartes. Dette er hvordan man tildeler en statisk IP-adresse til en beholder. 

Overskriv den nåværende NIC'en:

```lxc config device override <navn på beholder> <navn på NIC>```

Gi beholderen den statiske IP-adressen og omstart:

```lxc config device set <navn på beholder> <navn på NIC> ipv4.address <IP-adresse>```

```lxc restart <navn på beholder>```

## Eksponere beholderen for internett

Etter all denne konfigurasjonen er det fremdeles ikke mulig å aksessere beholderen fra internett. Du må derfor lage en ```iptables```-regel slik at nettverkstrafikk blir videresendt til beholderen. 

Dette krever 2 ting: Din offentlige IP-adresse og beholderen sin IP-adresse. 

### Installere Nginx

En god måte å teste om du har fått til iptables-regelen, er å installere Nginx i beholderen. Først må man inn i beholderen. Det kan man gjøre ved å skrive ```lxc shell <navn på beholder>```. Da "ssh"-er du inn i beholderen.

Du kan også skrive ```lxc exec <navn på beholder> -- sh -c "<kommandoer>"``` om du bare ønske å utføre kommandoer i beholderen. Deretter installerer du Nginx på den konvensjonelle måten for OS-en du har valgt. for Ubuntu blir det bare: 

```sh
apt-get update && apt-get install nginx
```

### Åpne for nettverkstrafikk

Dette steget er ganske avhengig av din lokale nettverks-konfigurasjon, men om du ikke har noen spesielle
nettverkskonfigurasjoner, burde dette fungere:

```sh
PORT=80 PUBLIC_IP=<offentlig IP-adresse> CONTAINER_IP=<beholder IP-adresse> IFACE=<navn på NIC> \
sudo -E bash -c 'iptables -t nat -I PREROUTING -i $IFACE -p TCP -d $PUBLIC_IP \
--dport $PORT -j DNAT --to-destination $CONTAINER_IP:$PORT -m comment --comment "<valgfri kommentar>"' \
```

Forklaring:

- ```-t nat``` spesifiserer at du skal bruke NAT-tabellen for adresse-oversettelse
- ```-I PREROUTING``` spesifiserer at du legger regelen til lenken som heter "PREROUTING"
- ```-i $FACE``` spesifiserer NIC'en som skal brukes
- ```-p TCP``` spesifiserer at TCP-protokollen skal brukes
- ```-d $PUBLIC_IP``` spesifiserer IP-adressen som er destinasjonen for regelen
- ```--dport $PORT``` spesifiserer porten som er destinasjonen for regelen
- ```-j DNAT``` spesifiserer at du skal utføre et "hopp" til destinasjons-NAT (DNAT)
- ```-to-destination $CONTAINER_IP:$PORT```spesifiserer at henvendelsen skal bli sendt til beholderen sin IP-adresse på den spesifikke porten. 

For å se en nummerert liste over reglene, kan du skrive ```sudo iptables -t nat -L PREROUTING --line-numbers```.

Disse reglene må man anvende på nytt hver gang man omstarter maskinen. Ganske kjedelig. Så da kan du installere ```iptables-persistent```-pakken. Da anvendes reglene automatisk hver gang du omstarter maskinen. 

For å fjerne en IP-tabell regel, kan du skrive ```sudo iptables -t nat -D PREROUTING <regelnummer>```. I tillegg burde du lagre forandringen ved å skrive ```sudo netfilter-persistent save```. Da anvendes ikke regelen på nytt igjen ved neste omstaI tillegg

## TL;DR

Installere og konfigurere LXD på en Ubuntu 20.04

1. ```sudo apt-get update && sudo apt-get upgrade```
2. ```sudo adduser <brukernavn> lxd```
3. ```sudo apt-get install -y zfsutils-linux```
4. ```sudo lxd init```
  - no
  - yes
  - <navn på storage pool>
  - zfs
  - yes
  - yes
  - /path/to/<navn på lagringsplass>
  - No
  - yes
  - lxdbr0
  - auto
  - auto
  - no
  - yes
  - no
5. ```lxc launch <os-navn>:<os-versjon> <beholder-navn>```
6. ```lxc config device override <beholder-navn> <NIC-navn>```
7. ```lxc config device set <beholder-navn> <NIC-navn> ipv4.address <beholder IP-adresse>```
8. ```lxc restart <beholder-navn>```
9. ```lxc exec <beholder-navn> -- sh -c "apt-get update && apt-get upgrade && apt-get install nginx"
10. ```PORT=80 PUBLIC_IP=<offentlig IP-adresse> CONTAINER_IP=<beholder IP-adresse> IFACE=<navn på NIC>  sudo -E bash -c 'iptables -t nat -I PREROUTING -i $IFACE -p TCP -d $PUBLIC_IP --dport $PORT -j DNAT --to-destination $CONTAINER_IP:$PORT -m comment --comment "<valgfri kommentar>"'```
11. ```sudo apt-get install iptables-persistent```
12. profitt

### Shell-script

Skap og start en ny beholder:

```sh
#!/bin/sh

lxc list --columns ns4

read -p "Name of OS? 'OS:version':" "osName"
read -p "Name of container?:" "containerName"
read -p "IP of container?:" "containerIP"
read -p "Name of NIC:" "nicName"
read -p "Servers public IP-address?:" "publicIP"
read -p "Port to receive network traffic?:" "portNr"

lxc launch $osName $containerName

lxc config device override $containerName $nicName
lxc config device set $containerName $nicName ipv4.address $containerIP

lxc restart $containerName

read -p "Forward incoming connections to container? [yes/no]:" "fwdTraffic"

if [ $fwdTraffic = "yes" ]; then

        read -p "Comment for Ip-table rule?:" "iptComment"

        PORT=$portNr PUBLIC_IP=$publicIP CONTAINER_IP=$containerIP IFACE=$nicName \
        sudo -E bash -c 'iptables -t nat -I PREROUTING -i $IFACE -p TCP -d $PUBLIC_IP \
        --dport $PORT -j DNAT --to-destination $CONTAINER_IP:$PORT -m comment --comment "$iptComment"'
	sudo netfilter-persistent save
	echo "Added rule to IP-table. The container is now ready."
else
        echo "Skipped forwarding network traffic. The container is now ready."
fi
```

## (Valgfritt) Installere LXDUI

Jeg har ikke prøvd det selv enda, men kommer til å oppdatere denne instruksjonen når jeg har implementert det.
Du kan også installere et visuelt brukergrensesnitt i nettleseren, med 
[dette github-prosjektet](https://github.com/AdaptiveScale/lxdui/wiki). 

## Kilder

Ser du noe galt med denne instruksjonen? [Opprett en pull request!](https://github.com/Cengelsen/cengelsen.no)

- [How to install and Configure LXD on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-lxd-on-ubuntu-20-04), sist lest 24.07.2022.

- [LXD > Getting started > Installation](https://linuxcontainers.org/lxd/getting-started-cli/), sist lest 24.07.2022

- [Difference between LXC and LXD](https://www.geeksforgeeks.org/difference-between-lxc-and-lxd/), sist lest 25.07.2022
- [Comparing LXD vs. LXC](https://discuss.linuxcontainers.org/t/comparing-lxd-vs-lxc/24), sist lest 25.07.2022
