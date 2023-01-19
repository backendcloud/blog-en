release time :2022-09-08 18:53


#Overlay2
Linux provides a file system called the Union File System, which has the following characteristics:
* Combined mount: Combine multiple directories hierarchically and mount them together to a joint mount point.
* Copy-on-write: Modifications to the joint mount point will not affect multiple underlying directories, but use other directories to record the modified operations.




OverlayFS is a stacked file system, which depends on and builds on other file systems (such as ext4fs and xfs, etc.), and does not directly participate in the division of disk space structure, but only divides the different directories in the original underlying file system into " Merge", and then present it to the user, which is the joint mount technology.

At present, there are many file systems that can be used as a joint file system to achieve the above functions: overlay2, aufs, devicemapper, btrfs, zfs, vfs, etc. And overlay2 is the file system currently recommended by docker: https://docs.docker.com/storage/storagedriver/select-storage-driver/

overlay2 includes lowerdir, upperdir and merged three levels, of which:
* lowerdir: Indicates the lower-level directory. Modifying the joint mount point will not affect lowerdir.
* upperdir: Indicates the upper-level directory. Modifying the joint mount point will be modified synchronously in upperdir.
* merged: It is the joint mount point after lowerdir and upperdir are merged.
* workdir: Used to store temporary files and indirect files after mounting.

# Docker experiments

    [root@centos7 kubevirt]# docker run -d -p 5000:5000 --restart=always --name registry registry:2
    Unable to find image 'registry:2' locally
    Trying to pull repository docker.io/library/registry ... 
    2: Pulling from docker.io/library/registry
    213ec9aee27d: Pull complete 
    5299e6f78605: Pull complete 
    4c2fb79b7ce6: Pull complete 
    74a97d2d84d9: Pull complete 
    44c4c74a95e4: Pull complete 
    Digest: sha256:83bb78d7b28f1ac99c68133af32c93e9a1c149bcd3cb6e683a3ee56e312f1c96
    Status: Downloaded newer image for docker.io/registry:2
    e9a3182ce19660264984b9c057e5cb01f8f3c478e515c3401117a856555bda74
    [root@centos7 kubevirt]# docker ps
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
    e9a3182ce196        registry:2          "/entrypoint.sh /e..."   21 seconds ago      Up 21 seconds       0.0.0.0:5000->5000/tcp   registry
    [root@centos7 kubevirt]# mount|grep overlay
    /dev/mapper/centos-root on /var/lib/docker/overlay2 type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
    overlay on /var/lib/docker/overlay2/f408abdbfe2c3fe90f84d50f16028bdcb3d865f7df23cce976bca77fc18e831c/merged type overlay (rw,relatime,context="system_u:object_r:container_file_t:s0:c88,c409",lowerdir=/var/lib/docker/overlay2/l/OLAQ7MQOXXNBFCPAZASEMGFRDT:/var/lib/docker/overlay2/l/OZ67SUB6WSDZ4WQDST6VRSAEFY:/var/lib/docker/overlay2/l/B52RB5YUGJXWBNNH7VOBZQZBLI:/var/lib/docker/overlay2/l/4ALOBU24OQJXJVNXCA5U52PCSV:/var/lib/docker/overlay2/l/3RDVYPKAIOMN6N6AU4CKFKWLTJ:/var/lib/docker/overlay2/l/2BWTKA2E4FN6DCMNHDMNAV5PRL,upperdir=/var/lib/docker/overlay2/f408abdbfe2c3fe90f84d50f16028bdcb3d865f7df23cce976bca77fc18e831c/diff,workdir=/var/lib/docker/overlay2/f408abdbfe2c3fe90f84d50f16028bdcb3d865f7df23cce976bca77fc18e831c/work)

can be seen:

* Joint mount point merged: /var/lib/docker/overlay2/f408abdbfe2c3fe90f84d50f16028bdcb3d865f7df23cce976bca77fc18e831c/merged
* lowerdir: /var/lib/docker/overlay2/l/OLAQ7MQOXXNBFCPAZASEMGFRDT:/var/lib/docker/overlay2/l/OZ67SUB6WSDZ4WQDST6VRSAEFY:/var/lib/docker/overlay2/l/* B52RB5YUGJXWBNNH7VOBZQZBLI:/var/lib/docker/overlay2/ l/4ALOBU24OQJXJVNXCA5U52PCSV:/var/lib/docker/overlay2/l/3RDVYPKAIOMN6N6AU4CKFKWLTJ:/var/lib/docker/* overlay2/l/2BWTKA2E4FN6DCMNHDMNAV5PRL
* upperdir: /var/lib/docker/overlay2/f408abdbfe2c3fe90f84d50f16028bdcb3d865f7df23cce976bca77fc18e831c/diff
* workdir: /var/lib/docker/overlay2/f408abdbfe2c3fe90f84d50f16028bdcb3d865f7df23cce976bca77fc18e831c/work

# Manual mount experiment

    [root@centos7 tt]# tree
    .
    ├── lower1
    │   ├── a
    │   └── b
    ├── lower2
    │   └── a
    ├── merged
    ├── upper
    │   └── c
    └── work
        └── work

    6 directories, 4 files
    [root@centos7 tt]# mount -t overlay overlay -o lowerdir=lower1:lower2,upperdir=upper,workdir=work merged
    [root@centos7 tt]# mount|grep overlay
    /dev/mapper/centos-root on /var/lib/docker/overlay2 type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
    overlay on /var/lib/docker/overlay2/f408abdbfe2c3fe90f84d50f16028bdcb3d865f7df23cce976bca77fc18e831c/merged type overlay (rw,relatime,context="system_u:object_r:container_file_t:s0:c88,c409",lowerdir=/var/lib/docker/overlay2/l/OLAQ7MQOXXNBFCPAZASEMGFRDT:/var/lib/docker/overlay2/l/OZ67SUB6WSDZ4WQDST6VRSAEFY:/var/lib/docker/overlay2/l/B52RB5YUGJXWBNNH7VOBZQZBLI:/var/lib/docker/overlay2/l/4ALOBU24OQJXJVNXCA5U52PCSV:/var/lib/docker/overlay2/l/3RDVYPKAIOMN6N6AU4CKFKWLTJ:/var/lib/docker/overlay2/l/2BWTKA2E4FN6DCMNHDMNAV5PRL,upperdir=/var/lib/docker/overlay2/f408abdbfe2c3fe90f84d50f16028bdcb3d865f7df23cce976bca77fc18e831c/diff,workdir=/var/lib/docker/overlay2/f408abdbfe2c3fe90f84d50f16028bdcb3d865f7df23cce976bca77fc18e831c/work)
    overlay on /root/tt/merged type overlay (rw,relatime,seclabel,lowerdir=lower1:lower2,upperdir=upper,workdir=work)
    [root@centos7 tt]# for i in `ls merged`;do echo $i: `cat merged/$i`;done
    a: in lower1
    b: in lower1
    c: in upper

> It can be seen that from the merged perspective, the a file located in lower2 is overwritten by the a file located in lower1; the b file is located in lower1, and the c file is located in upper, conforming to the hierarchical structure from high to low upper->lower1->lower2.


Add a file d in the merged directory, and the upper directory will automatically add the file d accordingly.

    [root@centos7 tt]# touch merged/d
    [root@centos7 tt]# tree
    .
    ├── lower1
    │   ├── a
    │   └── b
    ├── lower2
    │   └── a
    ├── merged
    │   ├── a
    │   ├── b
    │   ├── c
    │   └── d
    ├── upper
    │   ├── c
    │   └── d
    └── work
        └── work

    6 directories, 9 files





