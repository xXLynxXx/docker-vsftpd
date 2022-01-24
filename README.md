# VSFTPd in Docker

A 🐋Docker container that implements the **VSFTPD** server and has the following features:

## Features

-   Based on the **latest** `Alpine` and `Ubuntu` base images;
-   `Multi-stage` image build resulting in a tiny production image;
-   Fixed _segmentation fault_ (`segfault`) causing container to stop;
-   Organized the work of `Virtual Users` using the PAM module `libpam-pwdfile`;
-   Applied the latest `patches` taken from the Ubuntu/Alpine repositories;
-   Configured to work as `Active/Passive` mode FTP;

---

## How to get started

To get started, you need to install `Docker` and `Docker Compose` on your machine before proceeding further. To do this, you can refer to the following Docker documentation for help:

-   [Install Docker Engine](https://docs.docker.com/engine/install/);
-   [Install Docker Compose](https://docs.docker.com/compose/install/);

---

## Example of preparation and launch

### 1. Create Local User FTP & Directory

We recommend setting up a dedicated **local FTP user account** on the Linux server (in the example, this is `ftpadmin`), under which all VSFTPd **Virtual Users** will work in the future.

```sh
root@server:~$ adduser --home /home/ftp --uid 1001 ftpadmin
```

The `/home/ftp` directory will store Virtual User directories (`$USER` in the VSFTPd conf. file).

From under the new user, create a Virtual User directory, for example `VirtUser`.

```sh
ftpadmin@server:~$ mkdir --mode 755 ~/VirtUser
```

### 2. Running a Docker container

After all the preparatory steps, you can proceed to build and run the VSFTPd Docker container.

### 2.1 via `Docker-CLI`

Let's Build the Docker Image. We have prepared some Dockerfile based on Alpine/Ubuntu OS. These build instructions are no different, only the way the package managers of each OS work.

We will use a build based on `Dockerfile.alpine` as an example, but you can also use `Dockerfile.ubuntu`.  
To correctly process the build context, go to the directory `.../images/vsftpd/`.

```sh
docker build \
--build-arg VSFTPD_VERSION=3.0.5 \
--build-arg UID=1001 \
--force-rm \
--tag vsftpd:latest \
--file Dockerfile.alpine ./
```

-   `VSFTPD_VERSION`: is an argument specifying which version of VSFTPd to use when building. The default value `VSFTPD_VERSION=3.0.4`
-   `UID`: this is an argument that specifies the UID of the user that was created earlier ([Section 1](#1-create-local-user-ftp--directory)). The default value `UID=1000`

After creating the image, you need to run the Docker container based on it with correctly mounted volumes, forwarded ports, and additional environment variables.

```sh
docker run --detach \
--name vsftpd \
--restart always \
--publish 21:21 --publish 20:20 --publish 40000-40010:40000-40010 \
--env TZ=Europe/Moscow \
--mount type=bind,src=$(pwd)/applications/vsftpd/vsftpd.conf,dst=/etc/vsftpd/vsftpd.conf \
--mount type=bind,src=$(pwd)/applications/vsftpd/vsftpd.log,dst=/var/log/vsftpd.log \
--mount type=bind,src=$(pwd)/applications/vsftpd/virtual-users,dst=/etc/vsftpd/virtual-users \
--mount type=bind,src=$(pwd)/applications/vsftpd/vsftpd.virtual,dst=/etc/pam.d/vsftpd.virtual \
--mount type=bind,src=/home/ftp,dst=/home/ftp \
vsftpd:latest
```

### 2.2 via `Docker-Compose`

In fact, all the commands above are combined in the Docker-Compose YAML file `docker-compose-ftp.yml`. Thus, with one command, you will create image and run a container with your configuration.

```sh
docker-compose -f docker-compose-ftp.yml up --detach
```

If you wish, you can supplement or change the instructions presented in this file.

> ❗❗❗ Note that `vsftpd.conf` and `vsftpd.virtual` must be owned by `Root`. Otherwise, VSFTPd will exit and return an error.
>
> ```sh
> chown root:root ./applications/vsftpd/vsftpd.conf ./applications/vsftpd/vsftpd.virtual
> ```

## SSL

By default, VSFTPd is configured to work **without SSL**.
If you want to set up a secure FTP server using SSL/TLS, you will need to do the following:

1. The SSL/TLS certificate itself is required. You can purchase a TLS/SSL certificate from many CAs or generate a self-signed one using `OpenSSL` (we recommend using a `CFSSL` toolkit to create a complete PKI);

2. Add the `certificate` and `private-key` to the Container, for example through the Docker `bind-mount` mechanism.

```sh
docker run --detach \
--name vsftpd \
--restart always \
--publish 21:21 --publish 20:20 --publish 40000-40010:40000-40010 \
--env TZ=Europe/Moscow \
--mount type=bind,src=$(pwd)/applications/vsftpd/vsftpd.conf,dst=/etc/vsftpd/vsftpd.conf \
--mount type=bind,src=$(pwd)/applications/vsftpd/vsftpd.log,dst=/var/log/vsftpd.log \
--mount type=bind,src=$(pwd)/applications/vsftpd/virtual-users,dst=/etc/vsftpd/virtual-users \
--mount type=bind,src=$(pwd)/applications/vsftpd/vsftpd.virtual,dst=/etc/pam.d/vsftpd.virtual \
--mount type=bind,src=/home/ftp,dst=/home/ftp \
--mount type=bind,src=$(pwd)/vsftpd.crt,dst=/etc/vsftpd/cert/vsftpd.crt,readonly \
--mount type=bind,src=$(pwd)/vsftpd.key,dst=/etc/vsftpd/cert/vsftpd.key,readonly \
vsftpd:latest
```

Or add lines in `docker-compose-ftp.yml` to the `volumes:` directive. Example lines are contained in the file as comments;

3. Change the `ssl_enable=YES` option in `vsftpd.conf`

## Generate Virtual Users account

Since in our case the PAM module `libpam-pwdfile` is used, it is necessary to create account data in the `/etc/passwd` file format.

In order to create a virtual user account, you can use the [Apache htpasswd](https://httpd.apache.org/docs/2.4/programs/htpasswd.html) utility, which will create a string in the format "`login:password-hash`".

```sh
htpasswd -Bbn VirtUser password >> virtual-users
```

or through Docker

```sh
docker run --rm xmartlabs/htpasswd VirtUser password >> virtual-users
```

## Passive mode VSFTPd

For the correct operation of the **Passive Mode** of FTP, you must perform some actions:

1. If you plan to use the **Docker network driver**: `bridge` (attached by default), then your own subnet is created, with its own range of IP addresses.

    In this case, it is necessary to correct the configuration in `vsftpd.conf` under the heading "`-- Passive mode --`", namely, to replace the IP in the `pasv_address` option with the **IP of the host**, so that the client, when requesting a PASV command, returns the IP of the host, not Docker.

    > **Example**: Let's assume your server's **public IP** is `81.250.250.12` and the Docker container's `network:bridge` IP is `172.18.0.2`.
    > When a client sends a `PASV` command to an ftp server, it returns `172.18.0.2 + <random port in range>`. The client will try to connect to the specified socket, but will not be able to.
    > To do this, you need to change the option in `vsftpd.conf`, for the correct mapping:
    >
    > `pasv_address = 81.250.250.12`
    >
    > After that, the FTP server will return its **public IP** to the request and the client will continue communicating with FTP without any problems.

2. You can use **Docker network driver**: `host` - which will remove the network isolation between the container and the Docker host and use the host network directly.

    - Through the **Docker CLI**, append the argument to the main command:

    ```sh
    docker run --detach \
    --name vsftpd \
    --network host \
    --restart always \
    --env TZ=Europe/Moscow \
    --mount type=bind,src=$(pwd)/applications/vsftpd/vsftpd.conf,dst=/etc/vsftpd/vsftpd.conf \
    --mount type=bind,src=$(pwd)/applications/vsftpd/vsftpd.log,dst=/var/log/vsftpd.log \
    --mount type=bind,src=$(pwd)/applications/vsftpd/virtual-users,dst=/etc/vsftpd/virtual-users \
    --mount type=bind,src=$(pwd)/applications/vsftpd/vsftpd.virtual,dst=/etc/pam.d/vsftpd.virtual \
    --mount type=bind,src=/home/ftp,dst=/home/ftp \
    vsftpd:latest
    ```

    - Either via **Docker-Compose** by changing `docker-compose-ftp.yml` with the following content:

    ```yaml
    version: "3.8"
    services:
        # vsFTPd
        vsftpd:
            build:
                context: images/vsftpd
                dockerfile: Dockerfile.alpine # || Dockerfile.ubuntu
                args:
                    VSFTPD_VERSION: 3.0.5
                    UID: 1001
            image: vsftpd:latest
            container_name: vsftpd
            restart: always
            network_mode: "host" # Docker Host network
            volumes:
                - ./applications/vsftpd/vsftpd.conf:/etc/vsftpd/vsftpd.conf # vsftpd configuration file
                - ./applications/vsftpd/vsftpd.log:/var/log/vsftpd.log # LOG file
                - ./applications/vsftpd/vsftpd.virtual:/etc/pam.d/vsftpd.virtual # vsftpd PAM authentication rules
                - ./applications/vsftpd/virtual-users:/etc/vsftpd/virtual-users # virtual users from htpasswd generator
                # - ./vsftpd.crt:/etc/vsftpd/cert/vsftpd.crt # SSL certificate for FTPS
                # - ./vsftpd.key:/etc/vsftpd/cert/vsftpd.key # SSL private-key for FTPS
                - /home/ftp:/home/ftp # Working directory
            environment:
                TZ: "Europe/Moscow"
    ```

This will eliminate the need to map the container IP address to the host. Also comment out the `pasv_address` option in `vsftpd.conf` to avoid errors.

> ❗❗❗ Note that if you enable `PASV` mode, the internal ports used for passive mode (i.e. `pasv_min_port` to `pasv_max_port` ports in the config file `vsftpd.conf`) must be mapped to external ports (the same ports). For example, if `pasv_min_port=40000` and `pasv_max_port=40010`, you should add the parameter to Docker CLI "`--publish 40000-40010:40000-40010`" or to `docker-compose-ftp.yml`.

---

# Additional Features

For correct operation, a small CLI utility `patcher` was written, which allows you to transparently apply all the necessary patches.

You can find information about the utility by typing:

```sh
./images/vsftpd/patches/patcher --help
```

And if you don't need any `Ubuntu`/`Alpine`/`Docker` patches, change its mode in the `Dockerfile`.
