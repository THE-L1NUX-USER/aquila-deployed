 **Note : Choosing the AWS as the cloud platform.**
## Step 1 : Setting up an EC2 instance on AWS

1. Log in to the AWS Management Console.
2. In the navigation pane, choose `EC2`.
3. Choose `Launch Instance`.
4. Select the `Ubuntu 20.04 LTS AMI`.
5. Select the instance type `t2.medium` with 2 vCPU and 4GB RAM.
   
   This is the minimum settings told in the documentation of `AquilaCMS` 
6. Configure Instance Details.
7. Add Storage.
8. Add Tags.
9. Configure Security Group.
10. Create a new security group and add the following rules:
    - Type: HTTP, Protocol: TCP, Port Range: 80, Source: Anywhere
    - Type: HTTPS, Protocol: TCP, Port Range: 443, Source: Anywhere
    - Type: SSH, Protocol: TCP, Port Range: 22, Source: My IP
    - Type: Custom UDP, Protocol: UDP, Port Range: 51820, Source: Anywhere
11. Scroll down to `Key Pairs` and choose `Create a new key pair`.
12. Provide a name for the key pair, for example, `my-key-pair` and choose Download Key Pair.
13. Save the key pair file, `my-key-pair.pem`, in a secure location on your local machine.
14. Choose `Review and Launch`.

## Step 2 : SSH into the EC2 instance
15. Open a terminal window on your local machine and move to the path where the key pair file is stored.
16. Use the following command to connect to the EC2 instance, replacing my-key-pair.pem with the path to your key pair file, and `IP_ADDRESS` with the public IP address of your EC2 instance:
```bash
ssh -i "my-key-pair.pem" ubuntu@IP_ADDRESS
```
Once connected, you should see a command prompt for the EC2 instance.

## Step 3 : Installing Docker on the EC2 instance

17. Run the following command to update the repository:

```bash
sudo apt update
```
18. Create a new file called install-docker.sh using the nano editor:

```bash
nano install-docker.sh
```
19. Add the following contents to the file:
```bash
#!/bin/bash
echo "Checking if docker is installed..."

status=$(systemctl is-active docker.service)

if [ $status == "active" ]; then
    echo "Docker is running, stopping and removing it..."
    sudo systemctl stop docker.socket
    sudo systemctl stop docker.service
    sudo apt-get remove -y docker docker-engine docker.io containerd runc
    echo "Docker has been stopped and removed."
else
    echo "Docker is not running or not installed."
fi

echo "Updating apt package manager..."
sudo apt-get update -y

echo "Adding Docker's official GPG key..."
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg --yes

echo "Adding Docker's official repository to apt sources..."
sudo echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
echo "Updating apt package list..."
sudo apt-get update -y

echo "Installing Docker..."
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
sudo apt-get autoremove -y
sudo apt-get clean
echo "Adding current user to the Docker group..."
sudo usermod -aG docker $USER
echo "Starting Docker service..."
sudo systemctl start docker.service
sudo systemctl start docker.socket
```
Save the file using `ctrl+s` and exit using `ctrl+x`.

*Note: The `install-docker.sh` script is made by me to install Docker on Ubuntu.*

20. Run the following command to make the file executable:
```bash
sudo chmod +x install-docker.sh
```
21. Run the file using the following command:
```bash
sudo bash install-docker.sh
```
## Step 4 : Configure the domain:

22. Head over to your Cloudflare account where you have already configured the nameservers for the domain "attree.ml".

23. Click the DNS settings of the the domain "attree.ml".

24. Add a new "A" record with entry `@` add the `IP address` of the EC2 instance as the value and keep the proxy status to DNS only.

## Step 5 : Installing Nginx and Configuring Certbot:

25. Install Nginx using the following command:
```bash
sudo apt install nginx -y
```
26. To install Certbot, refer to the instructions on this link [certbot](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal)

Run the following command to obtain the SSL certificate:
```bash
sudo certbot certonly --nginx -m email@email.com -d attree.ml
```
*Note: Replace "email@email.com" with your own email address for important notifications about your SSL certificate.*

## Step 6 : Nginx Configuration

We will be configuring Nginx to work with our application.

27. Removing the Default Configuration

First, we will remove the default configuration files for Nginx by running the following commands:

```bash
sudo rm /etc/nginx/sites-available/default
sudo rm /etc/nginx/sites-enabled/default
```
28. Creating a New Configuration File
Next, we will create a new configuration file for our application by running the following command:

```bash
sudo nano /etc/nginx/sites-available/aquila.conf
```
Then, enter the following configuration into the file:

```bash
server {
    server_name _;

    listen [::]:443 ssl; 
    listen 443 ssl; 
    ssl_certificate /etc/letsencrypt/live/attree.ml/fullchain.pem; 
    ssl_certificate_key /etc/letsencrypt/live/attree.ml/privkey.pem; 
    return 301 http://attree.ml/;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    return 301 http://attree.ml/;
}

server {
    server_name attree.ml;

    listen [::]:443 ssl; 
    listen 443 ssl; 
    ssl_certificate /etc/letsencrypt/live/attree.ml/fullchain.pem; 
    ssl_certificate_key /etc/letsencrypt/live/attree.ml/privkey.pem; 
    include /etc/letsencrypt/options-ssl-nginx.conf; 
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; 

    location /admin {
        add_header X-Real-IP $remote_addr;
        allow 1.2.3.4; #change it to your VPN IP
        deny all;
        proxy_pass http://0.0.0.0:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        error_page 403 /forbidden.html;
    }

    location = /forbidden.html {
        internal;
    }

    location / {
        proxy_pass http://0.0.0.0:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

server {
    if ($host = attree.ml) {
        return 301 https://$host$request_uri;
    } 

    listen 80 ;
    listen [::]:80 ;

    server_name attree.ml;
    return 404; 
}
```
Save and exit the editor.

## **Note** : change the 1.2.3.4 to your VPN IP
For me it is going to be same as my EC2 instance IP as I am running a wireguard VPN on the server.

29. Creating a Forbidden Page
We will now create a new file named forbidden.html which will be displayed if the admin page is viewed without using a VPN.

```
sudo nano /usr/share/nginx/html/forbidden.html
````
And put the following content into the file:

```HTML
<!DOCTYPE html>
<html>
<head>
    <title>Forbidden</title>
    <style>
        body {
            text-align: center;
        }
        h1 {
            text-align: center;
        }
        h2 {
            text-align: left;
            font-size: 20px;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <h2>FORGOT YOU'R IP ?!!!</h2>
    <h1>NO ACCESS!!!</h1>
</body>
</html>
```
Save and exit the file.

30. Enabling the Configuration File
To enable our configuration file, we will use the following command:

```bash
sudo ln -s /etc/nginx/sites-available/aquila.conf /etc/nginx/sites-enabled/aquila.conf
```
31. Testing the Configuration
To test our configuration file, we will use the following command:

```bash
sudo nginx -t
```
If the configuration is valid, Nginx will display a message saying configuration test is successful.

32. Reloading Nginx
After testing the configuration, we will restart Nginx to apply the changes:

```bash
sudo systemctl reload nginx
```
Now the Nginx is configured to work with our application.

## Step 7 : Docker setup  

33.  Create a directory called aquila
```
mkdir aquila
```
34. Move into that directory
```bash
cd aquila
```
35. Create a file called docker-compose.yml
```bash
nano docker-compose.yml
```
36. Add the following content to the file:
```docker
version: '3'
services:
    aquila:
        image: aquilacms/aquilacms
        container_name: aquilacms
        restart: unless-stopped
        hostname: aquilacms
        depends_on:
          - mongo
        ports:
            - 3000:3010
        volumes:
           - "./aquilacms-data:/src" 
        networks:
            - aquila-network
    mongo:
       image: mongo
       container_name: mongo
       restart: unless-stopped
       hostname: mongodb
       ports:
            - 27017:27017
       volumes:
           - "./mongo-db:/data/db"      
       networks:
            - aquila-network
    wireguard:
     image: linuxserver/wireguard:latest
     container_name: wireguard
     hostname: wireguard
     cap_add:
      - NET_ADMIN
      - SYS_MODULE
     environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - SERVERURL=1.2.3.4 # Change IP address to the EC2 instance
      - SERVERPORT=51820 
      - PEERS=10 
      - PEERDNS=auto 
      - INTERNAL_SUBNET=10.13.13.0 
      - ALLOWEDIPS=0.0.0.0/0 
      - LOG_CONFS=true 
     volumes:
      - ./config:/config
      - /lib/modules:/lib/modules
     ports:
      - 51820:51820/udp
     sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
     restart: always
     networks:
            - aquila-network

networks:
  aquila-network:
    name : aquila
    driver: bridge
```
 Save and exit the file

## **Note** : change the 1.2.3.4 to EC2 instance IP

37. Run the containers using the following command:
```bash
docker compose up -d
```
## Step 8 :  Connecting to Wireguard VPN

38. In the same directory `aquila`, you will see a new folder named `config` created alongside the `docker-compose.yml` file. This `config` folder contains the configuration files for all the available peers.

39. To move into the `config` folder, use the following command:
```bash
cd config/
```
40. Inside the `config` folder, you will see 10 different peers available. Each peer has its own unique configuration file. 
It is important to note that only one client can use one peer at a time.

41. To select a specific peer, navigate to the peer's directory using the following command:

For example,select `peer1.
```bash
cd peer1/
```
42. Once you are inside the peer's directory, you will see the configuration file for that specific peer. This file is needed to connect to the VPN using the Wireguard client.

To install and use Wireguard, refer to this link: 
https://www.wireguard.com/install/

By using the configuration file of the peer, you can now connect to the VPN.
Please note that you will need to use the configuration file for the specific peer that you have selected.

## Step 9 :  Setting up the AquilaCMS initial install

43. Open up the website https://attree.ml make sure that the `VPN` is `CONNECTED`.

This is how the initial setup will look like.

![](https://media.discordapp.net/attachments/853458017738555403/1065695505272819792/image.png?width=1170&height=540 "")

Hit `next`.

44. Now fill the details as 
    - `Site Name` : AquilaCMS *(or anything)*.
    - `Path for the configuration` : *leave as default*.
    - `Connection string to MongoDB` : 
    ``` 
    mongodb://mongo:27017/database
    ```
    - `Website URL` : *leave as default*.
    - `Adminstration prefix` : admin

Also fill up `Adminstrative details` and choose `Default Language`.
 
 Hit `Save Configuration`.

 45. Now wait for the application to configure.

 **Note** : If got any `Gateway Timeout` error make sure that `VPN` is connected and `IP addesse` are configured properly in the configuration.

 46. After configuration is done we will be asked restart
 
 To do that we will restart the `aquila` docker container:

 ```bash
 docker restart aquila
 ```
47. After restart we will be greeted with our website: https://attree.ml

![](https://media.discordapp.net/attachments/853458017738555403/1065735102279188551/image.png?width=1170&height=536 "")

48. Admin page can be acessed on :  https://attree.ml/admin

![](https://media.discordapp.net/attachments/853458017738555403/1065735732909584465/image.png?width=1170&height=530 "")

**Note**: This page will be only accessible if connected through `VPN`

49. Viewing the admin page without connecting to VPN will redirect to `forbidden.html` page

![](https://media.discordapp.net/attachments/853458017738555403/1065737200119054406/image.png?width=1170&height=587)
