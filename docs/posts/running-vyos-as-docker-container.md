---
authors: [stimmerman]
comments: true
categories:
  - Containerlab
  - VyOS
date: 
  created: 2023-09-06
  updated: 2023-12-14
tags:
  - containerlab
  - docker
  - vyos
---

# Running VyOS as docker container
I recently started to use [Containerlab] and wanted to build a lab using [VyOS] routers. I searched a bit on the topic and found some scattered resources describing the process [^1][^2][^3]. This post is combining and adding to that information.

<!-- more -->
!!! note "Prerequisites and versions"
    This post assumes you have docker, git, squashfs-tools installed on your machine.

## Building LTS and rolling release ISO's
First build both the LTS and rolling ISO's from which the containers will be created. Create a directory for the build and execute the commands for one or both releases:
```bash
mkdir vyos-builds && cd vyos-builds
```

=== "LTS (1.3.4)"

    ```bash
    git clone -b equuleus --single-branch https://github.com/vyos/vyos-build vyos-lts
    docker run --rm -it --privileged -v $(pwd)/vyos-lts:/vyos -w /vyos vyos/vyos-build:equuleus bash
    ./configure --architecture amd64 --build-by "red9-homelab" --build-type release --version 1.3.4
    sudo make iso
    exit
    ```

=== "Rolling (1.4.x)"

    ```bash
    git clone -b sagitta --single-branch https://github.com/vyos/vyos-build vyos-rolling
    docker run --rm -it --privileged -v $(pwd)/vyos-rolling:/vyos -w /vyos vyos/vyos-build:sagitta bash
    sudo make clean
    sudo ./build-vyos-image iso --architecture amd64 --build-by "red9-homelab"
    exit
    ```

## Building the containers
When the build process is completed we can mount the ISO's and create the containers from it

=== "LTS (1.3.4)"

    ```bash
    mkdir lts-rootfs
    sudo mount -o loop vyos-lts/build/vyos-1.3.4-amd64.iso lts-rootfs
    mkdir lts-unsquashfs
    sudo unsquashfs -f -d lts-unsquashfs/ lts-rootfs/live/filesystem.squashfs
    # Fix locale
    sudo sed -i 's/^LANG=.*$/LANG=C.UTF-8/' lts-unsquashfs/etc/default/locale
    # Reducing the container size
    sudo rm -rf lts-unsquashfs/boot/*.img
    sudo rm -rf lts-unsquashfs/boot/*vyos*
    sudo rm -rf lts-unsquashfs/boot/vmlinuz
    sudo rm -rf lts-unsquashfs/lib/firmware/
    sudo rm -rf lts-unsquashfs/usr/lib/x86_64-linux-gnu/libwireshark.so*
    sudo rm -rf lts-unsquashfs/lib/modules/*amd64-vyos
    # Pack it up and import in docker
    sudo tar -C lts-unsquashfs -c . | sudo docker import - vyos:1.3.4 --change 'CMD ["/sbin/init"]'
    ```

=== "Rolling (1.4.x)"

    ```bash
    mkdir rolling-rootfs
    sudo mount -o loop vyos-rolling/build/vyos-1.4-rolling-*-amd64.iso rolling-rootfs
    mkdir rolling-unsquashfs
    sudo unsquashfs -f -d rolling-unsquashfs/ rolling-rootfs/live/filesystem.squashfs
    # Fix locale
    sudo sed -i 's/^LANG=.*$/LANG=C.UTF-8/' rolling-unsquashfs/etc/default/locale
    # Reducing the container size
    sudo rm -rf rolling-unsquashfs/boot/*.img
    sudo rm -rf rolling-unsquashfs/boot/*vyos*
    sudo rm -rf rolling-unsquashfs/boot/vmlinuz
    sudo rm -rf rolling-unsquashfs/lib/firmware/
    sudo rm -rf rolling-unsquashfs/usr/lib/x86_64-linux-gnu/libwireshark.so*
    sudo rm -rf rolling-unsquashfs/lib/modules/*amd64-vyos
    # Pack it up and import in docker
    export BUILD=$(date "+%y%m%d%H%M")
    sudo tar -C rolling-unsquashfs -c . | sudo docker import - vyos:1.4-$BUILD --change 'CMD ["/sbin/init"]'
    ```

If everything went according to plan there should be two VyOS docker images:
```bash
stimmerman@containerlab:~/vyos-builds$ docker image ls | grep vyos
vyos                                     1.4-202309061459   13036fe69138   About a minute ago   1.16GB
vyos                                     1.3.3              72ddbcdf7579   2 minutes ago        874MB
vyos/vyos-build                          current            74a186038e1e   13 days ago          3.07GB
vyos/vyos-build                          equuleus           b989e4dbc47a   2 months ago         4GB
```

## Running a VyOS container
With the containers build we can spin up an instance to check if it's working.

=== "Without persistent config"

    ```bash
    # Start the container
    docker run -d --name vyos-lts --privileged -v /lib/modules:/lib/modules vyos:1.3.3
    # Login to the router
    docker exec -ti vyos-lts su vyos
    ```

=== "With persistent config"

    ```bash
    # Create volume to hold the configuration
    docker volume create vyos-lts-config
    # Start the container
    docker run -d --name vyos-lts --privileged -v /lib/modules:/lib/modules -v vyos-lts-config:/opt/vyatta/etc/config vyos:1.3.3
    # Login to the router
    docker exec -ti vyos-lts su vyos
    ```

You should be greeted by a VyOS prompt like `vyos@vyos:/$` if all went well.

??? example "Enabling SSH Access"
    
    Exec into the container
    ```bash
    docker exec -ti vyos-lts su vyos
    ```
    Enable the SSH service in VyOS
    ```bash
    vyos@vyos:/$ conf
    vyos@vyos# set service ssh
    vyos@vyos# commit; save; exit
    ```
    Find out the IP address assigned to eth0 in my case 172.17.0.4
    ```
    vyos@vyos:~$ show int
    Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
    Interface        IP Address                        S/L  Description
    ---------        ----------                        ---  -----------
    eth0             172.17.0.4/16                     u/u
    lo               127.0.0.1/8                       u/u
    ```
    You should be able to SSH into the router now like `ssh vyos@172.17.0.4` using the password `vyos`

  [Containerlab]: https://containerlab.dev/
  [VyOS]: https://vyos.io/

[^1]: https://docs.google.com/document/d/1TUUVGLzetAX7_BIO6qtKDCC89j40eHa7bZrGiM5a3j8/
[^2]: https://forum.vyos.io/t/vyos-as-docker-container-journey/6128
[^3]: https://docs.vyos.io/en/latest/installation/virtual/docker.html