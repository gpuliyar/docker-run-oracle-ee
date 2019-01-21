# docker-run-oracle-ee
Run Docker Oracle Enterprise Edition

## What the document is about?
1. If you are familiar with Docker commands
2. You know about Oracle Database and it's enterprise edition
3. You want to do a quick installation of Oracle Database EE using Docker and bring up a Container so you can do your PoC work
4. For your project reasons, you need to expose the Oracle Service to be accessible from external machines other than the host machine
> Then this document will help you to set it up.

## Follow the below steps if you would like to run Oracle EE as a Docker Container
> Note: The steps will also help you to expose the Oracle service so you can access it from outside (other than localhost)
> Second note: I'm using Oracle Database Enterprise Edition 12.2.0.1

### First - pull the docker image
```bash
docker pull store/oracle/database-enterprise:12.2.0.1
```

### Second - create a docker volume to externalize the data
```bash
docker volume create oracle-db
```

### Third - run the Image as a Container
```bash
docker run -d -it --name oracleDB -v oracle-db:/ORCL -P store/oracle/database-enterprise:12.2.0.1
```
Let's try to understand what these input options mean:
1. `-d ->` Allows you to run the container in the background and print the Container ID
2. `-it ->` Instructs the Docker to allocate a psuedo-tty connected to the Container's stdin
3. `--name ->` assign a name to the Container
3. `-v ->` bind mount an existing volume
4. `-P ->` publish all exposed ports to random ports
5. And finally the image with tag info

## Now - post installation steps

### First connect to the Oracle Container sqlplus
```bash
docker exec -it oracleDB bash -c "source /home/oracle/.bashrc; sqlplus /nolog"
```

### Connect to the sys account as sysdba
> Note: The default password for `sys` account is `Oradoc_db1`. Ensure to change it as soon as you done setting up the container. You don't want to expose your services with the default password. 
> The default SID is ORCLCDB

```sql
connect sys/Oradoc_db1@ORCLCDB as sysdba
```

### Now alter the user `sys` and change the password
```sql
alter user sys identified by "*hello*there*"; // use any password as you like
```

### Then exit the sql
```bash
exit
```

### Now check if the sys password change
#### Login again
```bash
docker exec -it oracleDB bash -c "source /home/oracle/.bashrc; sqlplus /nolog"
```

#### Connect with the new password
```sql
connect sys/*hello*there*@ORCLCDB as sysdba
```

## Now, its time to expose to service so you can access from other machine
The database server exposes port 1521 for Oracle client connections over SQLNet protocol and port 5500 for Oracle XML DB. SQLPlus or any JDBC client can be used to connect to the database server from outside the container.

### First, find the mapped port and ip, by executing
```
docker ps 
```

> Sample result will look like:
```
user1@user-machine:~$ sudo docker ps
CONTAINER ID        IMAGE                                       COMMAND                  CREATED             STATUS                  PORTS                                              NAMES
9be8ab4b0685        store/oracle/database-enterprise:12.2.0.1   "/bin/sh -c '/bin/ba…"   40 hours ago        Up 40 hours (healthy)   0.0.0.0:32769->1521/tcp, 0.0.0.0:32768->5500/tcp   oracleDB
```

As you can see from the sample output from the above, the mapped port of `1521` is `32769`
> Note: The port might change in your setup. Use the port that shows up in your command

> Note: Now you need to use the bash so you can access the file system and change the tnsnames.ora file

### Connect to the Oracle Container bash
```bash
docker exec -it oracleDB bash -c "source /home/oracle/.bashrc; bash"
```

### Goto tnsnames.ora file location
```bash
cd $TNS_ADMIN
```

### Check if the file exits
```
[oracle@9be8ab4b0685 ORCLCDB]$ ls -lrt tnsnames.ora
-rw-r--r-- 1 oracle oinstall 396 Jan 20 05:36 tnsnames.ora
```

> Note: Sample out of the default file will look like below
```
[oracle@9be8ab4b0685 ORCLCDB]$ cat tnsnames.ora
ORCLCDB =   (DESCRIPTION =     (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))     (CONNECT_DATA =       (SERVER = DEDICATED)       (SERVICE_NAME = ORCLCDB.localdomain)     )   )
ORCLPDB1 =   (DESCRIPTION =     (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))     (CONNECT_DATA =       (SERVER = DEDICATED)       (SERVICE_NAME = ORCLPDB1.localdomain)     )   )
```

### Change the host and port
1. Change the host `0.0.0.0` to your host ip address
2. Change the port `1521` to the mapped port

### Do a tnsping to verify inside the container bash
```
tnsping orclcdb
```
> The above command will result in `OK`

## Your setup is complete!
