# Docker swarm stack for a self-hosted installation of CADLAB.io

This project provides a compose file and directory structure to run a self-hosted version of [CADLAB.io](https://cadlab.io).

**Table of contents**

## About CADLAB.io

[CADLAB.io](https://cadlab.io) is a git-based visual version control platform for collaborative PCB design. What makes CADLAB different from platforms like GitHub, GitLab, BitBucket and other software-oriented platforms, is that CADLAB is hardware-focused. It currently supports popular PCB design vendors like Altium, Autodesk Eagle, and KiCad. CADLAB works with native design files and renders schematics and PCB layouts right in a browser - no need to export, import or install anything. Just commit your design files to a repository, and you're good to go. One of the most powerful CADLAB features is interactive visual diff for schematics and PCBs. You can compare any two revisions of your design and see what exactly was changed from one version to another. Together with annotation feature, it enables hardware engineers to build a truly collaborative design process. Quality control, and peer-review of design haven't been easier. Please go explore more on our website https://cadlab.io.

CADLAB offers multiple options of where you can store your files. As every other version control platform, CADLAB provides cloud-based repositories hosting. It's free for open-source projects and there are free repositories for individuals. If you already using GitHub or GitLab (including self-hosted), you can easily connect CADLAB to your existing projects so that you continue keeping your files on those platforms, but can leverage CADLAB's graphical layer. And finally, for organization that can't keep their files elsewhere but their own server/network, there is a self-hosted version of CADLAB. You can deploy CADLAB to your own server as a stand-alone application or connected with a self-hosted GitLab. This project is meant to simplify the deployment process, and provides your with a pre-configured compose files to run CADLAB in a docker swarm. If you're not familiar with Docker, no worries, this documentation will provide you with a step-by-step guide on how to get it up and running.

If you're interested in running a self-hosted CADLAB instance, please get in touch with our [sales](https://cadlab.io/quiz).

## Installation

### System requirements

You can run CADLAB on a VPS or dedicated server. Below are recommended system requirements for a stand-alone installation of CADLAB and integration with GitLab.

**Stand-alone CADLAB installation:**
- 2-4+ CPU cores
- 6-8+ GB of RAM
- 40+ GB of storage
- Ubuntu/Debian/CentOS/Fedora operating system

**CADLAB integrated with a self-hosted GitLab:**
- 2-4+ CPU cores
- 4+ GB of RAM
- 20+ GB of storage
- Ubuntu/Debian/CentOS/Fedora operating system

**IMPORTANT:** There should be no other software on the server listening on ports 80, 443 and 22, as Nginx container needs to use these ports in order for CADLAB to function properly.

### Install Docker

Install the latest release of Docker following the official [instructions](https://docs.docker.com/engine/install/#server).

Run the following command to make sure Docker was installed correctly:  

```bash
docker version
```

The output of this command should be somewhat like this:  
![docker version](documentation/images/docker-version.png "Logo Title Text 1")

### Login into docker.cadlab.io

In order to pull images from CADLAB Docker registry you need to login using your Docker client and credentials you obtain with a Self-Hosted CADLAB.io. Run the following command in the terminal:

```bash
docker login docker.cadlab.io
```

The system will ask you for username/password and after entering them you should get a "Login Succeeded" message.

### Add DNS records

Add an A record for a domain/sub-domain you plan to use for CADLAB, for example:

```
cadlab.company.com
```

Additionally, add or modify an SPF record and include IP address of your CADLAB server. This step is needed in order for CADLAB to send emails. If you want CADLAB to send emails through your existing mail server, you can skip this step and specify custom mail server in the cadlab.json file. 

The process of adding an SPF record depends on your domain registrar and is usually well documented. But it all comes down to adding a TXT record with a value of the following format:

```
v=spf1 ip4:50.201.69.200 -all
```

Keep in mind to replace *50.201.69.200* with your IP address. If your domain already has SPF record, then you should only include the 'ip4:50.201.69.200' part to add your new server.

### Install git

This an optional step, but makes CADLAB update process more convenient. You can follow the official git installation instructions [here](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

### Get CADLAB swarm stack compose file

Clone this repository to your server or download the latest release from the [releases page]() on GitHub if you don't want to use git.

Place the files in the `/var/cadlab` or another directory of your choice.

```bash
git clone https://github.com/CADLAB-io/docker-swarm-stack.git /var/cadlab
```

Use the `master` branch to run CADLAB as a stand-alone application or checkout to `GitLab-backend`, if you'd like to integrate CADLAB with your self-hosted GitLab server.

```bash
cd /var/cadlab
git checkout GitLab-backend
```

### Init swarm

For better stability and security this installation should be run in a Docker swarm. To activate swarm mode run the following command:

```bash
docker swarm init
```

Currently, CADLAB.io supposed to be run on a single node, so you can ignore swarm join tokens.

### Create secrets

Docker swarm provides a secure mechanism for managing secrets, e.g. passwords or other sensitive information. We will use secrets to create a database password.

Create a text file with your password using vim or nano editor. Here is an example of how to do it with vim:

```bash
cd /var/cadlab
vi mysql_password.txt
```
Then type your password and save the file, press `Esc`, type `:wq`, and press `enter`.

Now we can create our Docker secret and delete the file with the password: 

```bash
docker secret create mysql_root_password mysql_password.txt
rm mysql_password.txt
```

You can also create a secret without creating the text file by executing the following line:

```bash
echo "your_secure_password" | docker secret create mysql_root_password -
```

But after that we recommend cleaning up your history, to make sure you password is not saved in plain text anywhere on the machine.

To delete your command from the history first run:

```bash
history
```

Locate your password creation password in the list and use its line number to delete it from the history:

```bash
history -d <N>
```

### Configure CADLAB

In order to config CADLAB you need to edit cadlab.json file, located in the configs directory. The settings file is in the JSON format and the very minimum you need to specify is a hostname:

```javascript
{
    "hostname": "cadlab.example.com"
}
```

Below is the list of settings you can specify.

#### hostname
A fully qualified domain name that you plan to use to access CADLAB.

#### automatic_backups
This setting controls if you want CADLAB to make automatic backups on a schedule. 

Possible values: 
- no
- daily
- weekly
- monthly

Backups are stored in the backups directory located in the swarm project. CADLAB will automatically rotate backups and keep 10 most recent backups.

#### ssl_tls_support
This settings controls if your CADLAB instance is going to be available over HTTP or HTTPS. By default HTTPS is disabled. `ssl_tls_support` is an object of the following structure:

```javascript
{
    ...
    "ssl_tls_support": {
        "enabled": true,
        "vendor": "letsencrypt"
    }
    ...
}
```

Below is the list of all object properties with available values:
- **enabled** - true/false enables HTTPS
- **vendor** - specifies what certificates to use. Possible values are
  - *letsencrypt* - CADLAB will automatically generate free Let's Encrypt certificates
  - *self-signed* - CADLAB will generate a custom Certificate Authority and TLS certificates for your domain. The Certificate Authority certificate will be placed in the certificates directory of the swarm project.
  - *external* - specifies that external certificates will be used for CADLAB. If this option is selected, then you need to place certificates in pem format and corresponding keys in the certificates directory of the swarm project. If you install CADLAB as a stand-alone application you need to provide two pairs of certificate/keys for the host name you specified in the `hostname` setting and `git.[hostname]`. For example, cadlab.example.com.pem/cadlab.example.com.key and git.cadlab.example.com.pem/git.cadlab.example.com.key.
- **custom_ca_key** - custom Certificate Authority key. You can specify this property if you want CADLAB to generate self-signed keys using your own Certificate Authority, or if you've chosen `external` in the vendor property and your certificates are signed with a custom CA.
- **custom_ca_pem** - custom Certificate Authority cert file in pem format. You can specify this property if you want CADLAB to generate self-signed keys using your own Certificate Authority, or if you've chosen `external` in the vendor property and your certificates are signed with a custom CA.

#### smtp
By default CADLAB uses a built-in send-only mail server. In order for this mail server to deliver emails successfully you need to add an SPF record as described in the [Add DNS records](#add-dns-records) section. If you prefer using your own mail server, you can provide SMTP connection info in this section. `smtp` is an object of the following structure:

```javascript
{
    ...
    "smtp": {
        "host": "example-mail-server.com",
        "port": 25,
        "username": "account_name",
        "password": "account_password",
    }
    ...
}
```

Below is the list of all object properties with available values:
- **host** - hostname of your mail server
- **port** - SMTP port
- **username** - username of the email account you going to use send emails through
- **password** - account password

### Add license file

Put your license.key file into configs directory of the swarm project. Do not modify or re-save the license file, as it will fail validation and license key will not be valid.

### Start CADLAB swarm

Now you're all good to deploy your swarm stack. From this swarm project directory run the following Docker command:

```bash
docker stack deploy --with-registry-auth -c stack.yml cadlab
```

Starting CADLAB for the first time will take quite some time, as Docker needs to download all the required images and perform initial installation. 

CADLAB is currently running 5 services:
- cadlab
- git-server
- mail-server
- mysql
- nginx

You can monitor the process of the installation buy using the following Docker command:

```bash
docker service ls
```



- Install docker
- Login to docker.cadlab.io
- create A records for cadlab and optional git hostname
- Add spa record in case internal mail server is used
- install git
- clone repo with stack file
- initiate swarm
- create secrets

- modify cadlab.json
- Add license
- deploy
  - view list of running services
  - view logs of container to identify issues

Updating CADLAB
Changing CADLAB configuration file
Backup & Restore
Clear Cache
CADLAB settings
    - hostname
    - ssl_tls
    - smtp


