 **Note : Choosing the AWS as the cloud platform.**
### Step 1 : Setting up an EC2 instance on AWS

1. Log in to the AWS Management Console.
2. In the navigation pane, choose `EC2`.
3. Choose `Launch Instance`.
4. Select the `Ubuntu 20.04 LTS AMI`.
5. Select the instance type `t2.micro` with 1 vCPU and 1GB RAM, which is eligible for the free tier.
6. Configure Instance Details.
7. Add Storage.
8. Add Tags.
9. Configure Security Group.
10. Create a new security group and add the following rules:
    - Type: HTTP, Protocol: TCP, Port Range: 80, Source: Anywhere
    - Type: HTTPS, Protocol: TCP, Port Range: 443, Source: Anywhere
    - Type: SSH, Protocol: TCP, Port Range: 22, Source: My IP
11. Scroll down to `Key Pairs` and choose `Create a new key pair`.
12. Provide a name for the key pair, for example, `my-key-pair` and choose Download Key Pair.
13. Save the key pair file, `my-key-pair.pem`, in a secure location on your local machine.
14. Choose `Review and Launch`.

### Step 2 : SSH into the EC2 instance
15. Open a terminal window on your local machine and move to the path where the key pair file is stored.
16. Use the following command to connect to the EC2 instance, replacing my-key-pair.pem with the path to your key pair file, and `IP_ADDRESS` with the public IP address of your EC2 instance:
```bash
ssh -i "my-key-pair.pem" ubuntu@IP_ADDRESS
```
Once connected, you should see a command prompt for the EC2 instance.

### Step 3 : Installing Docker on the EC2 instance

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
### Step 4 : Configure the domain:

22. Head over to your Cloudflare account where you have already configured the nameservers for the domain "thelinuxserver.cloud".
23. In the DNS settings, add a subdomain "aquila" to the domain "thelinuxserver.cloud" by creating a new "A" record.
24. In the "A" record, add the IP address of the EC2 instance as the value and keep the proxy status to DNS only.

### Step 5 : Installing Nginx and Configuring Certbot:

25. Install Nginx using the following command:
```bash
sudo apt -y install nginx 
```
26. To install Certbot, refer to the instructions on this link: (https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal)
Run the following command to obtain the SSL certificate:
```bash
sudo certbot certonly --nginx -m email@email.com -d aquila.thelinuxserver.cloud
```
*Note: Replace "email@email.com" with your own email address for important notifications about your SSL certificate.*

### Step 6 : Nginx Configuration

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
nano /etc/nginx/sites-available/aquila.conf
```
Then, enter the following configuration into the file:

```bash
server {
    server_name _;

    listen [::]:443 ssl; 
    listen 443 ssl; 
    ssl_certificate /etc/letsencrypt/live/aquila.thelinuxserver.cloud/fullchain.pem; 
    ssl_certificate_key /etc/letsencrypt/live/aquila.thelinuxserver.cloud/privkey.pem; 
    return 301 http://aquila.thelinuxserver.cloud/;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    return 301 http://aquila.thelinuxserver.cloud/;
}

server {
    server_name aquila.thelinuxserver.cloud;

    listen [::]:443 ssl; 
    listen 443 ssl; 
    ssl_certificate /etc/letsencrypt/live/aquila.thelinuxserver.cloud/fullchain.pem; 
    ssl_certificate_key /etc/letsencrypt/live/aquila.thelinuxserver.cloud/privkey.pem; 
    include /etc/letsencrypt/options-ssl-nginx.conf; 
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; 

    location /admin {
        add_header X-Real-IP $remote_addr;
        allow 3.111.215.11;
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
    if ($host = aquila.thelinuxserver.cloud) {
        return 301 https://$host$request_uri;
    } 

    listen 80 ;
    listen [::]:80 ;

    server_name aquila.thelinuxserver.cloud;
    return 404; 
}
```
Save and exit the editor.

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

```
sudo ln -s /etc/nginx/sites-available/aquila.conf /etc/nginx/sites-enabled/aquila.conf
```
31. Testing the Configuration
To test our configuration file, we will use the following command:

```
nginx -t
```
If the configuration is valid, Nginx will display a message saying configuration test is successful.

32. Reloading Nginx
After testing the configuration, we will restart Nginx to apply the changes:

```
sudo systemctl restart nginx
```
Now the Nginx is configured to work with our application.

