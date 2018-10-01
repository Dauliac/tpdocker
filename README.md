#TP 1 B3

```
    $systemctl start docker; systemctl enable docker
```
#Pourquoi pas en root ?
Pour authoriser un mec à utiliser docker sans etre root on utilise
```bash
    $usermod -aG docker user
    su - $USER
```
Cette commande ajoute le groupe docker à l'utilisateur sans modifier ses autres groupes.

Cela permet à un user d'utiliser le cli docker sans avoir toutes les permissions root, ce qui serait dangeureux. En effet on ne donne pas les permissions root à n'importe qui car un user root peut potentiellement détruire le système.

![root](http://memepeoplesuck.com/wp-content/uploads/2014/05/1400676453622.jpg)

##Ubuntu
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

tmpfs ou tempory file system est un système de fichiers en ram qui set à avoir un accès rapide en io sur des fichiers. On peut notement s'en servir pour du swap ou pour mounter le /tmp

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

We can curl the container from the vm's ip or from the `0.0.0.0` ip of the container
```bash
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
