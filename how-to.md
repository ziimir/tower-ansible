```
sudo apt update
sudo apt upgrade
```

### install neovim

`sudo apt install neovim`

### install zfs utils

`sudo apt install zfsutils-linux`

### install docker

https://docs.docker.com/engine/install/ubuntu/

### setup netplan

`sudo apt install network-manager`

```
sudo nvim /etc/netplan/00-installer-config-wifi.yaml

network:
  renderer: NetworkManager
```

### setup samba

`sudo apt install samba`

`sudo mkdir /mnt/tank` - создаем точку монтирования для zfs, samba ее и будет шарить

`sudo nvim /etc/samba/smb.conf`

```
[tank]
   comment = Tank Samba
   path = /mnt/tank
   read only = no
   browsable = yes
```

### create user and group for samba

`sudo groupadd sambausers`

`sudo useradd -s /bin/false familyman`

`sudo usermod -a -G sambausers familyman`

`sudo usermod -a -G sambausers ziimir` - себя тоже добавим

`sudo smbpasswd -a familyman` - добавляем нового пользака для smb

### make /mnt/tank writable

`sudo chgrp -R sambausers /mnt/tank/`

`sudo chmod -R g+rw  /mnt/tank/`

### configure zfs

create or import zfs file system

#### import

`sudo zpool import` - list pools

`sudo zpool import -a -f` - import all pools to pool configured mnt

`sudo zfs set sharesmb=on TANK` - [turn on smb for zfs](https://www.youtube.com/watch?v=G8btpRDLiTY)

### add acl for shared

`sudo apt install acl`

`sudo zfs set acltype=posixacl TANK`

`sudo setfacl -Rdm g:sambausers:rwx /mnt/tank/`

`sudo setfacl -Rm g:sambausers:rwx /mnt/tank/`

### configure docker containers tachidesk, deluge

`sudo vim /etc/systemd/system/tower-docker.service`

```
[Unit]
Description=tower docker services
PartOf=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/etc/docker/tower
ExecStart=/usr/bin/docker compose up
ExecStop=/usr/bin/docker compose down

[Install]
WantedBy=multi-user.target
```

`sudo mkdir /etc/docker/tower/`

`sudo vim /etc/docker/tower/docker-compose.yaml`

```
services:
  torrents:
    image: "lscr.io/linuxserver/deluge:latest"
    container_name: deluge
    environment:
      - PUID=1001 # familyman
      - PGID=1001 # sambausers
      - TZ=Europe/Moscow
      - DELUGE_LOGLEVEL=error
    volumes:
      - "/mnt/tank/torrents/config:/config"
      - "/mnt/tank/torrents/downloads:/downloads"
    ports:
      - "8112:8112"
      - "6881:6881"
      - "6881:6881/udp"
  manga:
    image: "ghcr.io/suwayomi/tachidesk:latest"
    container_name: tachidesk
    environment:
      - TZ=Europe/Moscow
    volumes:
      - "/mnt/tank/manga/tachidesk:/home/suwayomi/.local/share/Tachidesk"
    ports:
      - "4567:4567"
  media:
    image: "lscr.io/linuxserver/jellyfin:latest"
    container_name: jellyfin
    environment:
      - PUID=1001 # familyman
      - PGID=1001 # sambausers
      - TZ=Europe/Moscow
    volumes:
      - "/mnt/tank/configs/jellyfin:/config"
      - "/mnt/tank/anime:/data/anime"
    ports:
      - "8096:8096"
  monitoring:
    image: "nicolargo/glances:latest-full"
    container_name: glances
    environment:
      - GLANCES_OPT="-w"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/mnt/tank/configs/glances/glances.conf:/etc/glances.conf"
    ports: "61208-61209:61208-61209"
    pid: host
```

`sudo systemctl daemon-reload`

`sudo systemctl start tower-docker`

`sudo systemctl enable tower-docker`
