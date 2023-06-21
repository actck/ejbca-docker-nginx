# ejbca-docker-nginx
Steps to deploy EJBCA first without proxy, then deploy the proxy and EJBCA behind it in production like and on premise.

When i started my journey to deploy EJBCA CE in a production like environement, I wanted to use proxy directly to use it as fronted for EJBCA.
What I did not understand was that we need first to start EJBCA container once and generate the Management CA and the SuperAdmin.p12. 

With the SuperAdmin.p12, you can create the ROOT CA and SUB CA and then sign a Web TLS Certificate and and the .pem cert and .pem key to NGINX.

Management CA ---- signed ----> superadmin.p12 

Tree view of docker project:

```
├── data
├── docker-compose.yml
├── nginx
│──────── ├── certs
│──────── │──────── ├── management_CA.pem
│──────── │──────── ├── mgmt_cert.pem
│──────── │──────── └── mgmt_pvkey.pem
│──────── └── conf
│────────     └── ejbca.conf
```

- **data** will be the volume for the database, even if the container is deleted, the data will remain
- **docker-compose.yml** is the main configuration file for this project
- **nginx** is the folder that will contains, nginx configuration for EJBCA and the certificates

## Prerequisite for the host, here i'm on Rocky Linux 9:

- With root user:

a. Create user and change is home folder (it can be any folder)
```
adduser -m -d /opt pki
```

b. Add pki as owner of /opt
```
chown -R pki:pki /opt
```

c. Add a password
```
passwd pki
```

d. Add pki to sudoers
```
echo 'pki    ALL=(ALL)       ALL' | sudo EDITOR='tee -a' visudo
```

e. Log as pki user
```
su - pki
```

- With pki user:

a. update the host
```
sudo dnf update -y
```

b. Download repo and install docker
```
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
```

c. Add pki to docker group and start docker
```
sudo usermod -aG docker pki
#logout and login again, otherwise it will not be applied 
sudo systemctl enable docker
sudo systemctl start docker
```

d. Create tree folder project and go inside
```
mkdir -p docker/ejbca/{data,nginx/{certs,conf}}
cd docker/ejbca
```

## Initial configuration - EJBCA without proxy
To start with EJBCA and docker compose, I've followed the official documentation: https://doc.primekey.com/ejbca/tutorials-and-guides/tutorial-start-out-with-ejbca-docker-container

- Create the docker compose file

```
touch docker-compose.yml
```

Copy paste the content of this below:

<details>

<summary> docker-compose.yml</summary>

```
version: '3.9'

services:
  ejbca-database:
    container_name: ejbca-database
    image: mariadb:latest
    restart: always
    command: mysqld --character-set-server=utf8 --collation-server=utf8_bin --log-bin
    #check your user id with the command "id", applying ID user as owner on volume, otherwise systemd-coredump has ownership
    user: "1001:1001"
    networks:
      - database-bridge
    environment:
      - MYSQL_ROOT_PASSWORD=foo123
      - MYSQL_DATABASE=ejbca
      - MYSQL_USER=ejbca
      - MYSQL_PASSWORD=ejbca
    volumes:
      - ./data:/var/lib/mysql:rw

  ejbca-node1:
    hostname: ejbca-node1
    container_name: ejbca
    image: keyfactor/ejbca-ce:latest
    depends_on:
      - ejbca-database
    networks:
      - database-bridge
      - ejbca-bridge
    environment:
      - DATABASE_JDBC_URL=jdbc:mariadb://ejbca-database:3306/ejbca?characterEncoding=UTF-8
      - LOG_LEVEL_APP=INFO
      - LOG_LEVEL_SERVER=INFO
      - TLS_SETUP_ENABLED=true
      - DATABASE_USER=ejbca
      - DATABASE_PASSWORD=ejbca
      - PASSWORD_ENCRYPTION_KEY=changeit
      - CA_KEYSTOREPASS=changeit
      - EJBCA_CLI_DEFAULTPASSWORD=changeit
      - EJBCA_CLI_DEFAULT_USERNAME=ejbca
      - EJBCA_CLI_DEFAULT_PASSWORD=changeit
   ports:
      - "80:8080"
      - "443:8443"

networks:
  database-bridge:
    driver: bridge
  ejbca-bridge:
    driver: bridge
```
  
</details>

- Run the docker compose command
```
docker compose up -d
```

- Check the logs after it is started

```
docker compose logs -f
```

At the end of the deployment, you will see some indications on how to get the SuperAdmin.p12 certificate.
At the moment (21 June 2023) there is a bug and we can not enroll the Initial SuperAdmin through the UI ([check](https://github.com/Keyfactor/ejbca-ce/discussions/302#discussioncomment-6228311)) 

```
Health check now reports application status at /ejbca/publicweb/healthcheck/ejbcahealth
ejbca            *********************************************************************************************
ejbca            *                                                                                           *
ejbca            * A fresh installation was detected and a ManagementCA was created for your initial         *
ejbca            * administration of the system.                                                             *
ejbca            *                                                                                           *
ejbca            * Initial SuperAdmin client certificate enrollment URL (adapt port to your mapping):        *
ejbca            *                                                                                           *
ejbca            *   URL:      https://ejbca-node1:443/ejbca/ra/enrollwithusername.xhtml?username=superadmin *
ejbca            *   Password: Mft/2RdSOdf8nmaQAp9dRJBr                                                      *
ejbca            *                                                                                           *
ejbca            * Once the P12 is downloaded, use "Mft/2RdSOdf8nmaQAp9dRJBr" to import it.                  *
ejbca            *                                                                                           *
ejbca            *********************************************************************************************

```

On your host, check the ip address with `ip address` and on you computer with the web browser (firefox in my case).
Access to the URL below (which is the temporary url that allow enroll throug UI)

https://192.168.1.150/ejbca/enrol/keystore.jsp

![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/83df834b-1da4-4cd4-8468-2aa2242f3c3d)

Enter the username and password showed in the previous logs, in my example user: superadmin and password: Mft/2RdSOdf8nmaQAp9dRJBr

- Choose the key Specification according to the ManagementCA wich is create with RSA2048. Select ENDUSER.
![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/c14ba975-2746-458e-a5f1-9edc6af0c80a)

Click enroll, it will download the superadmin.p12.

Go to Security in Firefox > Certificates > Import and import it, type the password showed in the previous logs.

![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/ea9ed7c0-7015-4494-add5-e9e45f0d1e54)

And now you access again the PKI UI, a pop up displays, accept the risk

![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/6a943093-7655-4fe8-858e-8740889769e2)

And you have now access to the adminweb thanks to the SuperAdmin certificate.

![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/b2e67666-530e-4d10-a992-556c7f0125e9)

## Second step - Enabled NGINX 



