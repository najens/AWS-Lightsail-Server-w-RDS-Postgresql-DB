# AWS-Lightsail-Server-w-RDS-Postgresql-DB
Instructions to setup an AWS Lightsail Server with Apache2, Flask, and an RDS Postgresql DB

## Software Require)ments
- [Bash CLI with Git](https://git-scm.com/downloads)

## Generate Public/Private Key Pairs For Each Server User To Handle Secure Authentication
- Open Git Bash CLI (if you don't have Git Bash, you may need to install and use [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) to generate and save key pairs
- For each user, repeat the following steps using the CLI to generate Private/Public Key Pairs on your local machine
```
$ ssh-keygen
```
- Give the file a unique name identifying the user and save it in your .ssh directory on your local machine (on Windows this is /c/Users/<user>/.ssh/<unique_file_name>).
- Give the user a passhphrase  
    - The grader private key filename is **ud_grader**
    - The grader passhphrase is **udacity**

## Setup AWS Lightsail
- Sign into the [aws console](https://aws.amazon.com/lightsail/)
- Next to your name in the upper right corner of the screen, select the region you wish to use
- In AWS Services, selct **Lightsail** in the Compute category
- Click the **Create instance** button
- Make sure the Instance location matches the region you wish to use
- Select the **Linux/Unix** platform
- Under blueprints select **OS Only**
- Select **Ubuntu** (current version 16.04 LTS)
- Under SSH key pair manager select **Upload New**
- Upload the .pub key file for the root user located in your .ssh directory on your local machine
- Choose an instance plan
- Name your instance
- Select the **Create** button
- Once the instance is running select the project
- Click on the **Networking** tab
- Under Firewall click **Edit rules** and add the following custom ports
    - Custom TCP 2200 (SSH)
    - Custom TCP 5432 (POSTGRESQL)
    - Custom UDP 123 (NTP)
- In the top right corner of the screen click **Account**
- On the Account page click **Advanced**
- Under VPC Peering click **Enable VPC peering**

## Setup RDS PostgreSQL Database
### Create New DB Instance
- Go back to the AWS Console
- Under AWS services click **Relational Database Service** under the Database category
- Click on the **Instances** tab
- Click **Launch DB Instance**
- Select **PostgreSQL**
- Select **Next**
- Under Instance specifications use default settings and select **Free tier**
- Under Settings fill out the following:
    - DB instance identifier <your instance identifer>
    - Master username <**ubuntu**> (should match the Lightsail username)
    - Master password <your master password>
- Click **Next**
- Under Network & Security keep default settings, but change Public Accessibility to **Yes**
- Under Database options keep default settings but type in <**ubuntu**> for Database name to match PostgreSQL Master username and Lightsail username
- Keep all other settings at the default value unless you feel a need to change them such as the backup retention period
- Select **Launch DB instance**
- Click **Database instance details**
- Under the details section find the endpoint for future reference, you will use this to configure your SQLALCHEMY_DATABASE_URI and to connect to the database from your server or local machine via PG Admin. Each of these connections will require this endpoint, the port number 5432, the Master username, the Master password, and the database you wish to connect to.
  
### Add lighthouse VPC to RDS Security Group
- In the top left corner of your screen click on **Services**
- Click on **VPC** under the Networking & Content Deliver category   
- Under VPC resources click on **VPC Peering Connections**
- Copy the value in the Requester CIDRs column
- In the top left corner of the your screen click on **Services**
- Click on **EC2** under the Compute category
- Under Resources click on **Security Groups**
- Click on the **rds-launch-wizard** security group
- At the bottom of the screen click on **Inbound**
- Click on **Edit**
- Click on **Add Rule**
- Change the type to **PostgreSQL**
- Paste in the CIDR and click **Save**
  
## Log into Lightsail Ubuntu Machine from Bash CLI
- From the AWS Console go to Lightsail and open your instance
- You will use the Public IP and Username as well as your private key to log in
- Log in from the Bash CLI
```
$ ssh <Username>@<Public IP Address> -p 22 -i ~/.ssh/<private_key_file_name>
```
- If you receive a prompt that asks 'Are you sure you want to continue connecting?', type **yes**
- Enter the passphrase you created for the user

## Setup Uncomplicated Firewall
- Change the configured SSH port
```
$ sudo nano /etc/ssh/sshd_config
```
- Replace Port 22 with 2200 and save file
- Restart SSH
```
$ sudo /etc/init.d/ssh restart
```
- Type the following commands to setup the firewall. You will initially block all incoming and allow outgoing. Then you will allow the incoming ports you wish to use and enable the firewall.
```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 80/tcp
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 5432/tcp
$ sudo ufw allow 123/udp
$ sudo ufw enable
$ y
$ sudo ufw status
$ exit
```
- Go back to your Lightsail instance in AWS
- Under the Networking tab click on **Edit rules**
- Delete SSH TCP Port 22 and click **Save**
- Log back into the ubuntu machine using the new ssh port
```
$ ssh <Username>@<Public IP Address> -p 2200 -i ~/.ssh/<private_key_file_name>
```
## Install Packages
```
$ sudo add-apt-repository -y ppa:jonathonf/python-3.6
$ sudo apt-get update
$ sudo apt-get -y upgrade
$ sudo apt-get install finger
$ sudo apt-get install -y python3.6 python3.6-venv
$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.5 1
$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 2
$ sudo apt-get install -y python3-pip
$ sudo apt-get install -y apache2
$ sudo apt-get install libapache2-mod-wsgi-py3
$ sudo apt-get install -y postgresql postgresql-contrib
$ sudo apt-get install git
$ sudo apt-get autoremove -y
```

## Create New Ubuntu Users
- Repeat the following steps for each user
```
$ sudo adduser <username>
```
- Assign a password to the user (use <username> since we won't use password authentication)
- Give the user a name and any other details you want to fill out from the prompt
- In a different shell open the user's public key stored in their local .ssh folder and copy the contents
- Create a .ssh directory for the user
```
$ sudo mkdir /home/<username>/.ssh
```
- Create an authorized_keys file for the user
```
$ sudo nano /home/<username>/.ssh/authorized_keys
```
- Paste the user's public key and save the file
- Make the user the owner of the .ssh directory and authorized_keys file
```
$ sudo chown -R <username>:<username> /home/<username>/.ssh
```
- Set up permissions for .ssh directory and authorized_keys file
```
$ sudo chmod 700 /home/<username>/.ssh
$ sudo chmod 644 /home/<username>/.ssh/authorized_keys
```

### Give user sudo access
- Create a sudoers file for user
```
$ sudo nano /etc/sudoers.d/<username>
```
- Add the following line to the file and save it
```
grader ALL=(ALL) NOPASSWD:ALL
```

## Configure Timezone
- Type the following command and follow the prompt in the terminal
```
$ sudo dpkg-reconfigure tzdata
```

## SSH into Remote PostgreSQL DB from Ubuntu Machine
```
$ ssh -N -L 63333:<RDS Instance Endpoint>:5432 <lightsail username>@<public ip address> -p 2200 -i ~/.ssh/<private_key_file_name>
```

## Login to PostgreSQL from Ubuntu Machine
```
$ psql --host=<RDS instance endpoint> --port=5432 --username=<username> --password --dbname=<database> 
```
