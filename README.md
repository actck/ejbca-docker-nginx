# ejbca-docker-nginx
Steps to deploy EJBCA first without proxy, then deploy the proxy and EJBCA behind it in production like and on premise.

When i started my journey to deploy EJBCA CE in a production like environement, I wanted to use proxy directly to use it as fronted for EJBCA.
What I did not understand was that we need first to start EJBCA container once and generate the Management CA and the SuperAdmin.p12. And only after that we can setup nginx as frontend. 

With the SuperAdmin.p12, you can create the ROOT CA and SUB CA and then sign a Web TLS Certificate and and the .pem cert and .pem key to NGINX.

TL;DR

1. Prerequisites
2. Run EJBCA without Nginx
3. Creation of ROOT, SUB CA and TLS Profile and Certs
4. Run EJBCA with NGINX
5. Done




## Action to do before running anything

<details>
<summary> Prerequites for RHEL 9/Rocky Linux 9/Alma Linux 9</summary>

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
  
</details>



## Initial configuration - EJBCA without proxy
To start with EJBCA and docker compose, I've followed the official documentation: https://doc.primekey.com/ejbca/tutorials-and-guides/tutorial-start-out-with-ejbca-docker-container
**I did some tweaks compare to the tutorial, such as additionnal environment key, and TLS_SETUP_ENABLED with value true instead of simple**

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
      - TZ=Europe/Paris
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

### Enrolment
- Check the logs after it is started

```
docker compose logs -f
```

At the end of the deployment, you will see some indications on how to get the SuperAdmin.p12 certificate.
At the moment (21 June 2023) there is a bug and we can not enroll the Initial SuperAdmin through the UI ([check](https://github.com/Keyfactor/ejbca-ce/discussions/302#discussioncomment-6228311)) 

<details>

<summary> Logs showing username and password for SuperAdmin.p12 </summary>

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

</details>

#### STEP 1

On your host, check the ip address with `ip address` and on you computer with the web browser (firefox in my case).
Access to the URL below (which is the temporary url that allow enroll throug UI)

https://192.168.1.150/ejbca/enrol/keystore.jsp

Enter the username and password showed in the previous logs, in my example user: superadmin and password: Mft/2RdSOdf8nmaQAp9dRJBr

<details>
<summary> Click to view image of enrolment </summary>
  
  ![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/83df834b-1da4-4cd4-8468-2aa2242f3c3d)
  
</details>


#### STEP 2

- Choose the key Specification according to the ManagementCA wich is create with RSA 2048. Select ENDUSER.

Click enroll, it will download the superadmin.p12.

<details>
<summary> Click to view image of enrolment </summary>
  
  ![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/c14ba975-2746-458e-a5f1-9edc6af0c80a)
  
</details>


#### STEP 3

Go to Security in Firefox > Certificates > Import and import it, type the password showed in the previous logs.
<details>
<summary> Click to view image of enrolment </summary>
  
  ![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/ea9ed7c0-7015-4494-add5-e9e45f0d1e54)
  
</details>

#### STEP 4
And now you access again the PKI UI, a pop up displays, accept the risk


<details>
<summary> Click to view image of enrolment </summary>
  
  ![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/6a943093-7655-4fe8-858e-8740889769e2)
  
</details>

#### STEP 5
And you have now access to the adminweb thanks to the SuperAdmin certificate.
<details>
<summary> Click to view image of enrolment </summary>
  
  ![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/b2e67666-530e-4d10-a992-556c7f0125e9)

</details>

See next step of PKI Creation.

# Creation of ROOT, SUB CA and TLS Profile and Certs

Before continuing with NGINX as frontend for EJBCA, you will need to create your PKI Hierarchy.
The steps are commonly the same than the official documentation, I've just added some GIF procedure.

- [Create PKI Hierarchy](https://github.com/s0p4L1N/ejbca-docker-nginx/blob/main/tutorial-create-a-pki-hierarchy-in-ejbca.md)

- [Issue TLS Certificates](https://github.com/s0p4L1N/ejbca-docker-nginx/blob/main/tutorial-issue-tls-server-certificates-with-ejbca.md)

When you are finished with theses two tutorial, come back here.

# NGINX as Frontend

Now, you will deploy NGINX to use it as fronted for EJBCA.

1. Split the **PEM file** in two seperate file, one containing the **private key**, the other one containing the **certificates**.

- Recognize the difference
  - The private Key start by -----BEGIN PRIVATE KEY----- and end with -----END PRIVATE KEY-----
  - The certificates start by -----BEGIN CERTIFICATE----- and end with -----END CERTIFICATE-----

- The certificate file should look like below:
<details>
<summary> management_cert.pem </summary>

``` 
-----BEGIN CERTIFICATE-----
MIICijCCAjGgAwIBAgIUM3QOuvRix1FSWj2nLF6hHmSHURYwCgYIKoZIzj0EAwQw
NTEMMAoGA1UEBhMDVVNBMQwwCgYDVQQKDANVRk8xFzAVBgNVBAMMDkNvbXBhbnkg
U1VCIENBMB4XDTIzMDYyMjA5MTM1M1oXDTI0MDYyMTA5MTM1MlowMzEMMAoGA1UE
BhMDVVNBMQwwCgYDVQQKDANVRk8xFTATBgNVBAMMDE1hbmFnZW1lbnRDQTBZMBMG
ByqGSM49AgEGCCqGSM49AwEHA0IABIVV76Z0itpxB3ZLXYtT0Uz5XEXttpJTJbr/
53dLP2or9ehthtv002Pi9WbqQCbuuBy3w1mkQacUe06L0aQUTBGjggEfMIIBGzAf
BgNVHSMEGDAWgBQ4rMurXatxuv0QarkBneScOi8IMDBjBggrBgEFBQcBAQRXMFUw
LgYIKwYBBQUHMAKGImh0dHA6Ly9wa2kuaXNzLmxhbi9jZXJ0cy9TVUJDQS5jcnQw
IwYIKwYBBQUHMAGGF2h0dHA6Ly9wa2kuaXNzLmxhbi9vY3NwMBsGA1UdEQQUMBKC
C3BraS5pc3MubGFuggNwa2kwEwYDVR0lBAwwCgYIKwYBBQUHAwEwMgYDVR0fBCsw
KTAnoCWgI4YhaHR0cDovL3BraS5pc3MubGFuL2NybHMvU1VCQ0EuY3JsMB0GA1Ud
DgQWBBRaEnjEOMohdIsyQPm/2Zh+b9kViTAOBgNVHQ8BAf8EBAMCBaAwCgYIKoZI
zj0EAwQDRwAwRAIgeKUK+Qxz9d7CIH2zDK8s9eLoRGk3LXKxy2+zgkXueAICIAxC
gqD8gu9cBRE5tSjcYCK9Zn6we56iMVtjDKst1Y5y
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIICcjCCAhegAwIBAgIUM9x9Da0t7LyjyUzQ26gfpS3Im1kwCgYIKoZIzj0EAwQw
NjEMMAoGA1UEBhMDVVNBMQwwCgYDVQQKDANVRk8xGDAWBgNVBAMMD0NvbXBhbnkg
Uk9PVCBDQTAeFw0yMzA2MjExNDU5MDNaFw0zODA2MTcxNDU5MDJaMDUxDDAKBgNV
BAYTA1VTQTEMMAoGA1UECgwDVUZPMRcwFQYDVQQDDA5Db21wYW55IFNVQiBDQTBZ
MBMGByqGSM49AgEGCCqGSM49AwEHA0IABA1tyGpFMf6fNOXc3U87YIJtI3tL9ZA1
F5IABq7dRv4a8l58ZQwQLD9hD3AwvI+rBt7p8HGpGpH1RRVkMM3rI6ajggECMIH/
MBIGA1UdEwEB/wQIMAYBAf8CAQAwHwYDVR0jBBgwFoAUQfL+EFkiBb53zzGlPwjW
T6QW+rMwZAYIKwYBBQUHAQEEWDBWMC8GCCsGAQUFBzAChiNodHRwOi8vcGtpLmlz
cy5sYW4vY2VydHMvUk9PVENBLmNydDAjBggrBgEFBQcwAYYXaHR0cDovL3BraS5p
c3MubGFuL29jc3AwMwYDVR0fBCwwKjAooCagJIYiaHR0cDovL3BraS5pc3MubGFu
L2NybHMvUk9PVENBLmNybDAdBgNVHQ4EFgQUOKzLq12rcbr9EGq5AZ3knDovCDAw
DgYDVR0PAQH/BAQDAgGGMAoGCCqGSM49BAMEA0kAMEYCIQCNfsG1naKRNAF3GFnr
5rlj8QN2yM+up91P5RQHJI4W2wIhAKk8r+Fj2zXytjaSGp48S0vlm8qlaR2aaJLd
Awg8VWHu
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIB0zCCAXmgAwIBAgIUYBdw1t4l6/o7Q/a46ktDeYkYc+AwCgYIKoZIzj0EAwQw
NjEMMAoGA1UEBhMDVVNBMQwwCgYDVQQKDANVRk8xGDAWBgNVBAMMD0NvbXBhbnkg
Uk9PVCBDQTAgFw0yMzA2MjExNDUxNDBaGA8yMDUzMDYxMzE0NTEzOVowNjEMMAoG
A1UEBhMDVVNBMQwwCgYDVQQKDANVRk8xGDAWBgNVBAMMD0NvbXBhbnkgUk9PVCBD
QTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABP4vZhTG0SswcMLILySu68vSHvpW
aM2Ss6zPcW45pCehMrEh3o9khZm5dcywr6NwJXCcUw8mx4p+GGl2cucKUNKjYzBh
MA8GA1UdEwEB/wQFMAMBAf8wHwYDVR0jBBgwFoAUQfL+EFkiBb53zzGlPwjWT6QW
+rMwHQYDVR0OBBYEFEHy/hBZIgW+d88xpT8I1k+kFvqzMA4GA1UdDwEB/wQEAwIB
hjAKBggqhkjOPQQDBANIADBFAiBwS0KvQ90cYqAhbgWQxkLhvD+Wa1r4kDCy6v+J
gTZukQIhANzUUTrUobkbrm2WtQl7ckrTF+p5QqhwDa1wfClyRDrU
-----END CERTIFICATE-----

```
  
</details>

- The private key file should look like below:
<details>

<summary> management_pvkey.pem </summary>

```
-----BEGIN PRIVATE KEY-----
MIGTAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBHkwdwIBAQQgG0zrNE8M2tLk0fVJ
bqZILP58OWi6reT4ePhpN9Ri9+igCgYIKoZIzj0DAQehRANCAASFVe+mdIracQd2
S12LU9FM+VxF7baSUyW6/+d3Sz9qK/XobYbb9NNj4vVm6kAm7rgct8NZpEGnFHtO
i9GkFEwR
-----END PRIVATE KEY-----
```
  
</details>

- You will also need to paste the Management CA (internal CA) for Certificate Authentication, so NGINX will forward the certificate to the internal Apache server of EJBCA.

![Download_Management_CA](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/277c19ba-9c5a-45e5-82b7-d97704ec291c)

Now copy paste these three files into you nginx/certs folder on your host where the docker projet is. (`/opt/docker/ejbca/nginx/certs`)

- CA_Management.pem --> the internal 'ManagementCA' for Client Certificate Authentication verification
- management_cert.pem --> the TLS Server Certificate for the PKI itself (Nginx frontend)
- management_pvkey.pem --> the TLS Server private key

## Add NGINX ejbca conf

1. In `/opt/docker/ejbca/nginx/conf` create ejbca.conf file and copy paste the below configuration

<details>

<summary> ejbca.conf </summary>

```
server {
    listen 80;
    server_name pki.iss.lan;
    location / {
        proxy_pass                              http://ejbca-node1:8081/;
        proxy_set_header Host                   $http_host;
        proxy_set_header X-Real-IP              $remote_addr;
        proxy_set_header X-Forwarded-For        $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto      $scheme;

        return 301 https://$host$request_uri;

  }
}


server {
   listen 443 ssl;
   server_name pki.iss.lan;
   ssl_certificate     /etc/nginx/certs/management_cert.pem;
   ssl_certificate_key /etc/nginx/certs/management_pvkey.pem;


   ssl_client_certificate /etc/nginx/certs/CA_Management.pem;
   ssl_verify_client optional;


   location / {

   if ($ssl_client_verify != SUCCESS) {
     return 403;
   }

        proxy_pass                              http://ejbca-node1:8082/;
        proxy_set_header Host                   $http_host;
        proxy_set_header X-Real-IP              $remote_addr;
        proxy_set_header X-Forwarded-For        $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto      $scheme;
        proxy_set_header SSL_CLIENT_CERT        $ssl_client_cert;

    }

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains";


}
```
</details>

Now in the docker compose file

Here's the complete configuration with NGINX:

<details>

<summary> docker-compose.yml </summary>

```
version: '3.9'
services:
  ejbca-database:
    container_name: ejbca-database
    image: mariadb:latest
    restart: always
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
      - EJBCA_CLI_DEFAULTPASSWORD=ejbca
      - EJBCA_CLI_DEFAULT_USERNAME=ejbca
      - EJBCA_CLI_DEFAULT_PASSWORD=ejbca
      - TZ=Europe/Paris
      - PROXY_HTTP_BIND=0.0.0.0


  nginx:
    hostname: ejbca-proxy
    container_name: ejbca-proxy
    image: nginx
    depends_on:
      - ejbca-node1
    volumes:
      - ./nginx/conf/ejbca.conf:/etc/nginx/conf.d/ejbca.conf
      - ./nginx/certs:/etc/nginx/certs/
      - /run/docker.sock:/var/run/docker.sock:ro
    ports:
      - 80:80
      - 443:443
    restart: always
    networks:
      - proxy-bridge
      - ejbca-bridge


networks:
  database-bridge:
    driver: bridge
  ejbca-bridge:
    driver: bridge
  proxy-bridge:
    driver: bridge
```
  
</details>

What changes between old docker-compose and the new one for NGINX

- For the ejbca-node1 service:
  - addedd PROXY_HTTP_BIND=0.0.0.0 to the environment block to inform ejbca that there is a proxy as frontend
  - removed the ports block as it should not open the port to the outside world
 
- Added NGINX service code block
- added nginx conf to proxypass to the container
- Added a network for NGINX

  Now, without removing the containers, just execute `docker compose up -d` it will create the nginx container and recreate ejbca container according to the new configuration.

```
[+] Running 8/8
 ✔ nginx 7 layers [⣿⣿⣿⣿⣿⣿⣿]      0B/0B      Pulled                                                                                                                                                      14.1s
   ✔ 5b5fe70539cd Pull complete                                                                                                                                                                          8.8s
   ✔ 441a1b465367 Pull complete                                                                                                                                                                         11.5s
   ✔ 3b9543f2b500 Pull complete                                                                                                                                                                         11.5s
   ✔ ca89ed5461a9 Pull complete                                                                                                                                                                         11.5s
   ✔ b0e1283145af Pull complete                                                                                                                                                                         11.6s
   ✔ 4b98867cde79 Pull complete                                                                                                                                                                         11.6s
   ✔ 4a85ce26214d Pull complete                                                                                                                                                                         11.6s
[+] Building 0.0s (0/0)
[+] Running 4/4
 ✔ Network ejbca_proxy-bridge  Created                                                                                                                                                                   0.5s
 ✔ Container ejbca-database    Running                                                                                                                                                                   0.0s
 ✔ Container ejbca             Started                                                                                                                                                                  18.1s
 ✔ Container ejbca-proxy       Started 
 ```








