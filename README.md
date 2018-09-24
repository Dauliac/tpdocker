# TP 1 B3

```
    $systemctl start docker; systemctl enable docker
```
## Pourquoi pas en root ?
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
On peut y voir les poits de montages, leurs droits, les devices, le `$PATH`


## Rkt:

```bash
    wget https://github.com/rkt/rkt/releases/download/v1.30.0/rkt-1.30.0-1.x86_64.rpm
    rpm -ivh rkt-1.30.0-1.x86_64.rpm
```


