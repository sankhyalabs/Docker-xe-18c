Optimized Oracle XE 18c on Docker
=====
Changes: 3rd May 2019: Docker image more optimized from 5.41 GB to 4.02 GB

Build a Docker image containing Oracle XE 18c with optimized size (4.02 GB instead of 8.7 GB) including Archivelog Mode and Flashback


## Oracle XE 18c Software
[Download Oracle XE 18c](https://jvtst-repository.s3.amazonaws.com/oracle-database-xe-18c-1.0-1.x86_64.rpm) `.rpm` file and drop it inside folder `18.4.0`.

### AWS AMI 
If you instantiate on amazon use this ami and follow these steps

```
AMI COD: ami-03383c16defb1ed02 | DESCRIPTION : RANCHEROS 1.5.4 
$ ros console list
$ sudo ros console switch ubuntu
```


## Prerequirements
You will need to complete following prerequirements
```
$ groupadd -g 54321 oinstall
$ cd /home
$ mkdir oracle
$ useradd -d /home/oracle -m -g oinstall -G docker -u 54321 oracle
$ chown -R oracle:oinstall oracle/
$ visudo
# Add --> oracle ALL=(ALL) NOPASSWD: ALL
# type it --> esc + :
# Save ---> wq!

# Depending on your disk layout you will need to create 4 directories
# e.g. you have one mount point called u01, create the following 4 directories below and switch the ownership to the oracle # user
$ chmod 777 /home/oracle/Docker-xe-18c/buildDockerImage.sh
$ mkdir -p /u01/diag
$ mkdir -p /u01/oradata
$ mkdir -p /u01/fast_recovery_area
$ mkdir -p /u01/tools
$ chown -R oracle:oinstall /u01/diag
$ chown -R oracle:oinstall /u01/oradata
$ chown -R oracle:oinstall /u01/fast_recovery_area
$ chown -R oracle:oinstall /u01/tools
```
## Build it
To build the images just run below command:
```
$ ./buildDockerImage.sh -v 18.4.0 -x
. . .
Successfully built ac71812fa9d3
Successfully tagged oracle/database:18.4.0-xe


  Oracle Database Docker Image for 'xe' version 18.4.0 is ready to be extended:

    --> oracle/database:18.4.0-xe

  Build completed in 697 seconds.

$ docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
oracle/database      18.4.0-xe           2018ad58d2f3        16 hours ago        4.02GB
```

## Run the Container
Just run following command, which will create your Oracle XE 18c Container, mount the internal directories `/opt/oracle/oradata`, `/opt/oracle/diag`, `/opt/oracle/fast_recovery_area` and `/opt/oracle/tools` to your prior created directories on your Docker host and enables a port forwarding of the Listener Port 1521 to 1521
```
docker run -d --name oraxe18c \
              -p 1521:1521 \
              -e ORACLE_PWD=[your password] \
              -e ORACLE_CHARACTERSET=[your characterset] \
              -e TZ=[your timezone] \
              -v [host directory for oradata]:/opt/oracle/oradata \
              -v [host directory for diag]:/opt/oracle/diag \
              -v [host directory for fast_recovery_area]:/opt/oracle/fast_recovery_area \
              -v [host directory for tools]:/opt/oracle/tools \
              [--network [your bridged network] \
              oracle/database:18.4.0-xe

# For Example
docker run -d --name oraxe18c \
              -p 1521:1521 \
              -e ORACLE_PWD=Oracle18c \
              -e ORACLE_CHARACTERSET=AL32UTF8 \
              -e TZ=Europe/Zurich \
              -v /u01/oradata:/opt/oracle/oradata \
              -v /u01/diag:/opt/oracle/diag \
              -v /u01/fast_recovery_area:/opt/oracle/fast_recovery_area \
              -v /u01/tools:/opt/oracle/tools \
              --network mynet \
              oracle/database:18.4.0-xe
```

After that you can create under [host directory for tools], e.g. `/u01/tools` a simple rman backup script will following content and perform a first full backup with RMAN
```
vi /u01/tools/full_bkp.rman

connect target sys/[your password]@XE
run {
    backup database plus archivelog;
}

# For Example
connect target sys/Oracle18c@XE
run {
    backup database plus archivelog;
}

# Or as compressed backupset plus delete backed up archivelogs
connect target sys/Oracle18c@XE
run {
    backup as compressed backupset database plus archivelog delete input;
}
```

To execute now a full backup simply run following docker command:
```
docker exec -it oraxe18c rman @/opt/oracle/tools/full_bkp.rman
```
In case you want to execute the RMAN Backup job from crontab replace `-it` with `-i`

