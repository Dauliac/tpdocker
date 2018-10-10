# TP 1 B3 RÉALISÉ PAR JULIEN DAULIAC & RAPHAEL BEGHIN

```
    $systemctl start docker; systemctl enable docker
```
# Pourquoi pas en root ?
Pour authoriser un mec à utiliser docker sans etre root on utilise
```bash
    $usermod -aG docker user
    su - $USER
```
Cette commande ajoute le groupe docker à l'utilisateur sans modifier ses autres groupes.

Cela permet à un user d'utiliser le cli docker sans avoir toutes les permissions root, ce qui serait dangeureux. En effet on ne donne pas les permissions root à n'importe qui car un user root peut potentiellement détruire le système.

![root](http://memepeoplesuck.com/wp-content/uploads/2014/05/1400676453622.jpg)

## Ubuntu
Pour cette partie nous irons vite car nous connaissons déja les commandes de base de docker
Pour lancer un container ubuntu on fait:
```bash
    $docker run -d -it ubuntu
```

pour les cartes réseaux el famoso iproute2:
```bash
    $apt update
    $apt install iproute2
    $ip a
```

On lance le slip:
```bash
    $docker exec -d container_name sleep 9000
```

Ici on voit met en évidence les namespaces:
```bash
    [root@localhost ~]# docker exec -it angry_noether ls -l /proc/10/ns/
    total 0
    lrwxrwxrwx. 1 root root 0 Sep 24 12:55 ipc -> 'ipc:[4026532127]'
    lrwxrwxrwx. 1 root root 0 Sep 24 12:55 mnt -> 'mnt:[4026532125]'
    lrwxrwxrwx. 1 root root 0 Sep 24 12:55 net -> 'net:[4026532130]'
    lrwxrwxrwx. 1 root root 0 Sep 24 12:55 pid -> 'pid:[4026532128]'
    lrwxrwxrwx. 1 root root 0 Sep 24 12:55 user -> 'user:[4026531837]'
    lrwxrwxrwx. 1 root root 0 Sep 24 12:55 uts -> 'uts:[4026532126]'

    [root@localhost ~]# ls -l /proc/1680/ns/
    total 0
    lrwxrwxrwx. 1 root root 0 Sep 24 14:54 ipc -> ipc:[4026532127]
    lrwxrwxrwx. 1 root root 0 Sep 24 14:54 mnt -> mnt:[4026532125]
    lrwxrwxrwx. 1 root root 0 Sep 24 14:54 net -> net:[4026532130]
    lrwxrwxrwx. 1 root root 0 Sep 24 14:54 pid -> pid:[4026532128]
    lrwxrwxrwx. 1 root root 0 Sep 24 14:54 user -> user:[4026531837]
    lrwxrwxrwx. 1 root root 0 Sep 24 14:54 uts -> uts:[4026532126]
```

Ici on peut voir les cgroups:
```bash
    cgtop
    Path                                                                          Tasks   %CPU   Memory  Input/s Output/
    ...
    /docker/4c02eeb8c16ecff428b76e6435391560a676beb8e2dfb2c8afe4fc327d0951f4          1      -   140.0K        -        -
    /docker/b07df25fe0a62170daea716b1eaa838b79ee779e51e679af0d336d67a49da9b0          1      -   136.0K        -        -
    /docker/eb2b15ad444f12c3457f0fcb15012d2b5cd1c10a74b5abd497aded722ef01685          2      -    75.7M        -        -
    ...
```

## tmpfs:

tmpfs ou tempory file system est un système de fichiers en ram qui set à avoir un accès rapide en io sur des fichiers. On peut notement s'en servir pour du swap ou pour mounter le /tmp ou l'hibernation

```bash
    docker volume create --driver local --opt type=tmpfs --opt device=tmpfs --opt o=size=100m,uid=1000 foo
    docker run -d -it --name devtest --mount source=foo,target=/app alpine
    docker container inspect devtest
        "Mounts": [
                {
                    "Type": "volume",
                    "Source": "foo",
                    "Target": "/app"
                }
```

## Runc
Ce fichier est une description d'un filesystem en `json`.
On peut y voir les points de montages, leurs droits, les devices, le `$PATH`

## Rkt:

```bash
    wget https://github.com/rkt/rkt/releases/download/v1.30.0/rkt-1.30.0-1.x86_64.rpm
    rpm -ivh rkt-1.30.0-1.x86_64.rpm
```

```bash
    systemd-run rkt --insecure-options=image --stage1-name=coreos.com/rkt/stage1-fly:1.29.0  run docker://alpine --exec /bin/sleep -- 9999
```

- on fait tourner `rkt` avec `systemd-run` pour créer une service unit
- `--insecure-options=image` `run docker://alpine` run le container depuis un registre docker avec une image alpine
- `exec /bin/sleep -- 9999` execute un sleep sur le container*

## Dockerfile

```bash
    if [ "$1" = '' ]; then
    chown -R postgres "$PGDATA"

    if [ -z "$(ls -A "$PGDATA")" ]; then
        gosu postgres initdb
    fi

    exec gosu postgres "$@"
    fi
    exec "$@"
```

On peut curl le container depuis l'ip de la vm ou en local depuis `0.0.0.0`

```bash
    docker run -d -it -p 1010:9999 601074aa1df6
    docker build --build-arg PYTHON_PORC=9999 .
    [root@localhost dockerfile]# docker ps
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
    ca8dadf7ab9e        601074aa1df6        "/bin/sh -c '/usr/bi…"   5 seconds ago       Up 4 seconds        0.0.0.0:1010->9999/tcp   sad_perlman
    [root@localhost dockerfile]# curl 0.0.0.0:1010
    <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
    <html>
    <head>
    <meta http-equiv="Content-Type" content="text/html; charset=ascii">
    <title>Directory listing for /</title>
    </head>
    <body>
    <h1>Directory listing for /</h1>
    <hr>
    <ul>
    <li><a href="test.html">test.html</a></li>
    </ul>
    <hr>
    </body>
    </html>
```

Voici notre Dockerfile:

```bash
    [root@localhost dockerfile]# cat Dockerfile
    FROM debian:latest

    ARG PYTHON_PORC=8888

    ENV PORT=${PYTHON_PORC}

    RUN mkdir /web && \
        apt-get update && apt-get upgrade && apt-get install python3 -y && \
        groupadd -g 999 nagini && \
        useradd -r -u 999 -g nagini nagini

    WORKDIR /web

    COPY test.html /web

    EXPOSE $PORT

    USER nagini:nagini

    ENTRYPOINT /usr/bin/python3 -m http.server $PORT
```

![nagini](https://i.imgflip.com/2j4bdp.jpg)

## Docker api:

Les containers:

```bash
    [root@localhost ~]# curl --unix-socket /var/run/docker.sock http:/containers/json | jq
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100   907  100   907    0     0   113k      0 --:--:-- --:--:-- --:--:--  147k
    [
      {
        "Id": "ca8dadf7ab9e3d26dd388a947ded6fd5356018cd0217173d5397acea1d0904fd",
        "Names": [
          "/sad_perlman"
        ],
        "Image": "601074aa1df6",
        "ImageID": "sha256:601074aa1df60133b976f031c4bfcfc6c5d2ced8296fa9a204af1b692c1e3b90",
        "Command": "/bin/sh -c '/usr/bin/python3 -m http.server $PORT'",
        "Created": 1538396154,
        "Ports": [
          {
            "IP": "0.0.0.0",
            "PrivatePort": 9999,
            "PublicPort": 1010,
            "Type": "tcp"
          }
        ],
        "Labels": {},
        "State": "running",
        "Status": "Up About an hour",
        "HostConfig": {
          "NetworkMode": "default"
        },
        "NetworkSettings": {
          "Networks": {
            "bridge": {
              "IPAMConfig": null,
              "Links": null,
              "Aliases": null,
              "NetworkID": "941ca1a31b075af17ddadf76ec836566ca7a51832b685ab95a265f6e8247fa51",
              "EndpointID": "35123ab1f45ed197585542792af71209304f5892a40d3378c03d1c5b9a0b0976",
              "Gateway": "172.17.0.1",
              "IPAddress": "172.17.0.2",
              "IPPrefixLen": 16,
              "IPv6Gateway": "",
              "GlobalIPv6Address": "",
              "GlobalIPv6PrefixLen": 0,
              "MacAddress": "02:42:ac:11:00:02",
              "DriverOpts": null
            }
          }
        },
        "Mounts": []
      }
    ]
```

Les images docker (on aime les jsons spam):

```bash
    [root@localhost ~]# curl --unix-socket /var/run/docker.sock http:/images/json | jq
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100  1647  100  1647    0     0   405k      0 --:--:-- --:--:-- --:--:--  536k
    [
      {
        "Containers": -1,
        "Created": 1538396101,
        "Id": "sha256:601074aa1df60133b976f031c4bfcfc6c5d2ced8296fa9a204af1b692c1e3b90",
        "Labels": null,
        "ParentId": "sha256:dbe11adb42cfa6a36aa3cfa50c71d262f6aa91bf0c8d9004c0f3e7cb98fe9022",
        "RepoDigests": [
          "<none>@<none>"
        ],
        "RepoTags": [
          "<none>:<none>"
        ],
        "SharedSize": -1,
        "Size": 158361209,
        "VirtualSize": 158361209
      },
      ...
      {
        "Containers": -1,
        "Created": 1536096084,
        "Id": "sha256:f2aae6ff5d896839bfb8609cb1510bcf36efcb6950683c3bcfb760668b0eefbe",
        "Labels": null,
        "ParentId": "",
        "RepoDigests": [
          "debian@sha256:07fe888a6090482fc6e930c1282d1edf67998a39a09a0b339242fbfa2b602fff"
        ],
        "RepoTags": [
          "debian:latest"
        ],
        "SharedSize": -1,
        "Size": 100576015,
        "VirtualSize": 100576015
      }
    ]
```

Pour obtenir l'ip du container:

```bash
[root@localhost ~]# curl --unix-socket /var/run/docker.sock http:/containers/sad_perlman/json | jq ".NetworkSettings.Networks.bridge.IPAddress"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4930    0  4930    0     0  2132k      0 --:--:-- --:--:-- --:--:-- 2407k
"172.17.0.2"
```

![localhost](https://proxy.duckduckgo.com/iur/?f=1&image_host=http%3A%2F%2Fblog.totaljobs.com%2Fwp-content%2Fuploads%2F2012%2F11%2Fip_meme.jpg&u=https://blog.totaljobs.com/wp-content/uploads/2012/11/ip_meme.jpg)

## Docker daemon:

### Secomp:

`secomp` est activé par défaut dans docker et il n'est pas recommandé de changer le profil par défaut.
Il s'agit d'un système de white list qui met en place un système de permission sur les appels systemes. Cette whiteliste se met en place sur `personality`, `socket`, `socketcall` et sur les autres variantes des appels système. - [source](https://docs.docker.com/engine/security/seccomp/#pass-a-profile-for-a-container)

```bash
    [root@localhost]# docker info
    ...
    Profile: /etc/docker/seccomp.json
    ...
```

### User namespace:

Install user namespaces:
```bash
    grubby --args="user_namespace.enable=1" --update-kernel=/boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
    reboot
    cat /etc/docker/daemon.json
    {
        "userns-remap": "vagrant"
    }
```

## Docker compose:

On sait déja faire: [exemple](https://github.com/Dauliac/cloud/tree/master/exemple)

Ah et le service systemd en cadeau:

```bash
[dauliac@ghostbuster ~]$ cat /etc/systemd/system/common_server.service
[Unit]
Description=Server service: traefik/nextcloud/mysql
After=network.target docker.service
[Service]
Type=simple
WorkingDirectory=/srv/common/
ExecStart=/usr/local/bin/docker-compose -f /srv/common/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /srv/common/docker-compose.yml down
Restart=always
[Install]
WantedBy=multi-user.target
```

## Portainer:
C'est comme la partie du dessus mais en plus facile.

Voici quand meme le `docker-compose`

```yaml
version: '3'

services:
  portainer:
    image: portainer/portainer
    command: -H unix:///var/run/docker.sock
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    network:
        - local

volumes:
  portainer_data:

networks:
  local:
    driver: bridge
```

## Fini RÉALISÉ PAR JULIEN DAULIAC & RAPHAEL BEGHIN:
![merci](https://pics.me.me/thank-you-so-much-13606464.png)

