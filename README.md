# Docker swarm stack for a self-hosted installation of CADLAB.io

This project provides a compose file and directory structure to run a self-hosted version of [CADLAB.io](https://cadlab.io).

**Table of contents**

- [Docker swarm stack for a self-hosted installation of CADLAB.io](#docker-swarm-stack-for-a-self-hosted-installation-of-cadlabio)
  - [About CADLAB.io](#about-cadlabio)
  - [Installation](#installation)
    - [System requirements](#system-requirements)
    - [Install Docker](#install-docker)
    - [Move sshd to another port (stand-alone installation only)](#move-sshd-to-another-port-stand-alone-installation-only)
    - [Login into docker.cadlab.io](#login-into-dockercadlabio)
    - [Add DNS records](#add-dns-records)
    - [Install git](#install-git)
    - [Get CADLAB swarm stack compose file](#get-cadlab-swarm-stack-compose-file)
    - [Init swarm](#init-swarm)
    - [Create secrets](#create-secrets)
    - [Configure CADLAB](#configure-cadlab)
      - [hostname](#hostname)
      - [automatic_backups](#automatic_backups)
      - [ssl_tls_support](#ssl_tls_support)
      - [smtp](#smtp)
      - [reverse_proxy](#reverse_proxy)
    - [Add license file](#add-license-file)
    - [Start CADLAB swarm](#start-cadlab-swarm)
    - [Checking services status](#checking-services-status)
  - [Troubleshooting](#troubleshooting)
    - [Checking container health](#checking-container-health)
    - [502 bad gateway](#502-bad-gateway)
  - [Changing CADLAB configurations](#changing-cadlab-configurations)
  - [Updating CADLAB](#updating-cadlab)
  - [Backup & restore CADLAB](#backup--restore-cadlab)
    - [Backing up CADLAB](#backing-up-cadlab)
    - [Restoring CADLAB](#restoring-cadlab)
  - [CADLAB command-line utility](#cadlab-command-line-utility)
  - [Integrating with GitLab](#integrating-with-gitlab)



## About CADLAB.io

[CADLAB.io](https://cadlab.io) is a git-based visual version control platform for collaborative PCB design. What makes CADLAB different from platforms like GitHub, GitLab, BitBucket, and other software-oriented platforms, is that CADLAB is hardware-focused. It currently supports popular PCB design vendors like Altium, Autodesk Eagle, and KiCad. CADLAB works with native design files and renders schematics and PCB layouts right in a browser - no need to export, import, or install anything. Just commit your design files to a repository, and you're good to go. One of the most powerful CADLAB features is interactive visual diff for schematics and PCBs. You can compare any two revisions of your design and see what exactly was changed from one version to another. Together with the annotation feature, it enables hardware engineers to build a truly collaborative design process. Quality control and peer-review of design haven't been easier. You can find more information on our website https://cadlab.io.

CADLAB offers multiple options of where you can store your files. Like every other version control platform, CADLAB provides cloud-based repositories hosting. It's free for open-source projects, and there are free repositories for individuals. If you are already using GitHub or GitLab (including self-hosted), you can easily connect CADLAB to your existing projects so that you continue keeping your files on those platforms but can leverage CADLAB's graphical layer. Finally, for organizations that can't keep their files elsewhere but their own server/network, there is a self-hosted version of CADLAB. You can deploy CADLAB to your own server as a stand-alone application or connected with a self-hosted GitLab. This project is meant to simplify the deployment process and provide you with pre-configured compose files to run CADLAB in a docker swarm. If you're not familiar with Docker, no worries. This documentation will provide you with a step-by-step guide on how to get it up and running.

If you're interested in running a self-hosted CADLAB instance, please contact our [sales](https://cadlab.io/quote).

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

**IMPORTANT:** There should be no other software on the server listening on ports 80, 443, and 22, as the Nginx container needs to use these ports for CADLAB to function properly.

### Install Docker

Install the latest release of Docker following the official [instructions](https://docs.docker.com/engine/install/#server).

Run the following command to make sure Docker was installed correctly:  

```bash
docker version
```

The output of this command should be somewhat like this:  

![docker version](documentation/images/docker-version.png "Docker version")

### Move sshd to another port (stand-alone installation only)

**Important**: This section is required only for the stand-alone CADLAB installation. If you install CADLAB with an external git back-end like GitLab, you should skip this section.

If you install CADLAB as a stand-alone application, it starts with a built-in git server that requires an SSH connection to interact with git repositories. Docker will proxy all SSH connections to the git container, making it impossible to connect to the host machine via SSH. We recommend moving the sshd server to another port to fix this issue, for example, 2222. To do this, you need to edit the sshd_config file.

SSH into your server and edit the following file `/etc/ssh/sshd_config` using `vi` or `nano` editor.

Somewhere close to the top of the file, you should see the following section:

```bash
# The strategy used for options in the default sshd_config shipped with
# OpenSSH is to specify options with their default value where
# possible, but leave them commented.  Uncommented options override the
# default value.

Include /etc/ssh/sshd_config.d/*.conf

#Port 22
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::
```

Uncomment the line `#Port 22` by removing the pound sign `#` and change `22` to `2222`. Now restart your sshd service by executing the following command:

```bash
service sshd restart
```

From now on, your sshd will be listening on port `2222`, and there will be no conflicts with Docker. To connect to your host machine, you will need to specify port like so:

```bash
ssh -p 2222 YOUR_USER@YOUR_IP
```

### Login into docker.cadlab.io

TO pull images from the CADLAB Docker registry, you need to log in using your Docker client and credentials you obtain with a Self-Hosted CADLAB.io. Run the following command in the terminal:

```bash
docker login docker.cadlab.io
```

The system will ask you for a username/password, and after entering them, you should get a "Login Succeeded" message.

### Add DNS records

Add an A record for a domain/sub-domain you plan to use for CADLAB, for example:

```
cadlab.company.com
```

If you install CADLAB as a stand-alone application, you also need to create an A record for the git service. It has to be the same domain/subdomain as for the CADLAB app, but prepended with `git.` like so:

```
git.cadlab.company.com
```

Additionally, add or modify an SPF record and include the IP address of your CADLAB server. This step is needed for CADLAB to send emails. If you want CADLAB to send emails through your existing mail server, you can skip this step and specify a custom mail server in the cadlab.json file. 

The process of adding an SPF record depends on your domain registrar and is usually well documented. But it all comes down to adding a TXT record with a value of the following format:

```
v=spf1 ip4:50.201.69.200 -all
```

**Important:** Keep in mind to replace *50.201.69.200* with your IP address. If your domain already has an SPF record, you should only include the 'ip4:50.201.69.200' part to add your new server.

### Install git

This an optional step but makes CADLAB update process more convenient. You can follow the official git installation instructions [here](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

### Get CADLAB swarm stack compose file

Clone this repository to your server or download the latest release from the [releases page](https://github.com/CADLAB-io/CADLAB.io-swarm-stack/releases/) on GitHub if you don't want to use git.

Place the files in the `/var/cadlab`.

```bash
git clone https://github.com/CADLAB-io/CADLAB.io-swarm-stack.git /var/cadlab
```

### Init swarm

For better stability and security, the CADLAB application should be launched in a Docker swarm. To activate swarm mode, run the following command:

```bash
docker swarm init
```

After running this command, you will see a message with a command you can use to add nodes to your swarm. Currently, CADLAB.io is supposed to be run on a single node, so you can ignore swarm join tokens. 

### Create secrets

Docker swarm provides a secure mechanism for managing secrets, e.g., passwords or other sensitive information. We will use secrets to create a database password. Below we explain how to create a secret using a text file or input redirection.

The first option is to create a secret using a text file. To continue with this option, create a text file with your password using vim or nano editor. Here is an example of how to do it with vim:

```bash
cd /var/cadlab
vi mysql_password.txt
```

Then press `i` to enter the editing mode and type your password. To save the file, press `Esc`, type `:wq`, and press `enter`.

Now we can create our Docker secret and delete the file with the password: 

```bash
docker secret create mysql_root_pass mysql_password.txt
rm mysql_password.txt
```

To create a secret using input redirection, you need to perform the following commands:

```bash
echo "your_secure_password" | docker secret create mysql_root_pass -
```

But after you create a secret using input redirection, we recommend cleaning up your history. This way you, can make sure your password is not saved in plain text anywhere on the machine.

To delete your command from the history first run:

```bash
history
```

Locate your secret creation line in the list and use its line number to delete it from the history:

```bash
history -d <N>
```

### Configure CADLAB

In order to configure CADLAB, you need to edit the cadlab.json file, located in the `/var/cadlab/configs` directory. The settings file is in the JSON format, and the very minimum you need to specify is a hostname:

```javascript
{
  "hostname": "cadlab.example.com"
}
```

To further configure CADLAB, you can add additional settings to the file. Below is the full list of settings you can specify.

#### hostname
A fully qualified domain name that you plan to use to access CADLAB.

```javascript
{
  "hostname": "cadlab.example.com"
}
```

#### automatic_backups
This setting controls automatic CADLAB backups. 

Possible values: 
- `no`
- `daily`
- `weekly`
- `monthly`

```javascript
{
  "hostname": "cadlab.example.com",
  "automatic_backups": "daily"
}
```

If this setting is not specified in the cadlab.json file, the default value is `no`.

Backups are stored in the `/var/cadlab/backups` directory located in the swarm project. CADLAB will automatically rotate backups and keep 10 most recent backups.

#### ssl_tls_support
This setting controls if your CADLAB instance is going to be available over HTTP or HTTPS. If this setting is not added, then HTTPS is disabled by default. `ssl_tls_support` is an object of the following structure:

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
- **enabled** - `true` or `false` enables/disables HTTPS. **Important:** if CADLAB is behind a [reverse proxy](#reverse_proxy), you may leave HTTPS disabled and handle HTTPS connection in your reverse proxy. If you want to keep HTTPS communication between your reverse proxy and CADLAB you must use an `external` SSL certificate that your host machine with reverse proxy trusts. You may also choose `self-signed`, but this will require adding a custom Certificate Authority (CA) generated by CADLAB to your host with reverse proxy.
- **vendor** - specifies what certificates to use. Possible values are:
  - `letsencrypt` - CADLAB will automatically generate free Let's Encrypt certificates. **IMPORTANT**: Let's Encrypt needs to be able to access your CADLAB installation to validate the certificates. If the server you install CADLAB on is not reachable from the internet, you need to choose the `external` or `self-signed option.
  - `self-signed` - CADLAB will generate a custom Certificate Authority (CA) and TLS certificates for your domain. The Certificate Authority certificate will be placed in the `certificates` directory of the swarm project. In order for your users to access the website, they will need to add the generated CA to their computers. The instructions on adding a custom CA depend on an operating system and are well covered on the internet.
  - `external` - specifies that external certificates will be used for CADLAB. If this option is selected, you need to place certificates in the pem format and corresponding keys in the `certificates` directory of the swarm project. If you install CADLAB as a stand-alone application, you need to provide two pairs of certificate/keys for the hostname you specified in the `hostname` setting and `git.[hostname]`. For example, `cadlab.example.com.pem` / `cadlab.example.com.key` and `git.cadlab.example.com.pem` / `git.cadlab.example.com.key`.
- **custom_ca_key** - custom Certificate Authority (CA) key. You can specify this property if you want CADLAB to generate self-signed keys using your own Certificate Authority or if you've chosen `external` in the vendor property and your certificates are signed with a custom CA.
- **custom_ca_pem** - custom Certificate Authority (CA) cert file in pem format. You can specify this property if you want CADLAB to generate self-signed keys using your own Certificate Authority or if you've chosen `external` in the vendor property and your certificates are signed with a custom CA.

#### smtp
By default, CADLAB uses a built-in send-only mail server. In order for this mail server to deliver emails successfully, you need to add an SPF record as described in the [Add DNS records](#add-dns-records) section. If you prefer using your own mail server, you can provide SMTP connection info in this section. `smtp` is an object of the following structure:

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
- **host** - hostname of your mail server.
- **port** - SMTP port. It depends on your mail server setup, but usually, 587 if an encrypted protocol is used, or 25 as the default SMTP port.
- **protocol** - (optional) encrypted protocol. Skip this setting if your server doesn't require secure connection. Possible values are:
  - `tls`
  - `ssl`
- **from** - email address to be used as a sender.
- **username** - username of the email account you are going to use to send emails through. This setting is required only if your SMTP server requires authentication.
- **password** - account password. This setting is required only if your SMTP server requires authentication.

#### reverse_proxy
CADLAB is deployed with its own Nginx container to handle incoming traffic for ports `80`, `443`, and, optionally, `22`. If you install CADLAB on a server with other software running on the same ports, you may want to use a reverse proxy to handle connections. This will require modifying `stack.yml` or `stack-external-git.yml` file, depending on whether you deploy CADLAB with git or connect it to GitLab.

To do so, you need to change port bindings in the Nginx service section to some other ports you want to use for CADLAB on your server. For example, `8080` instead of `80`, and `8443` instead of `433` like below:

```yaml
ports:
    - mode: host
      target: 22
      published: 22
      protocol: tcp
    - mode: host
      target: 80
      published: 8080
      protocol: tcp
    - mode: host
      target: 443
      published: 8443
      protocol: tcp
```

Then in your reverse proxy, redirect requests to the domain specified in `hostname` setting to the ports you specified in the stack file. In most cases, you would disable [ssl_tls_support](#ssl_tls_support) in CADLAB and proxy requests to port `80`, leaving  HTTPS handling for your reverse proxy. You may choose to use HTTPS connection for communication between your proxy and CADLAB for more information on this [read here](#ssl_tls_support).

In addition to configuring your proxy and changing the stack file, you also need to tell CADLAB it's behind a proxy. To do that, you need to add the following section to `cadlab.json` file:

```javascript
{
    ...
    "reverse_proxy": {
        "enabled": true,
        "protocol": "https"
    }
    ...
}
```

Below is the list of all object properties with available values:
- **enabled** - `true` or `false` - tells CADLAB is it's behind a reverse proxy.
- **protocol** - `http` or `https` - tells CADLAB which protocol is used to access CADLAB in the reverse proxy, so that CADLAB can build correct links to pages and resource files like css and js.

### Add license file

Put your `license.key` file into the `/var/cadlab/configs` directory of the swarm project. Do not modify or re-save the license file, as it will fail validation, and the license key will not be valid. 

You can copy the license file from your local machine to the server in multiple ways. For example, if you're on Mac or Linux, you can use the `scp` [command](http://www.hypexr.org/linux_scp_help.php) like so:

```
scp -P 2222 license.key YOUR_USER@YOUR_IP:/var/cadlab/configs
```
`-P` stands for the port. You will need to use this if you install CADLAB as a stand-alone application since CADLAB uses standard SSH port `22` to work with your git container. If you use an external git back-end like GitLab, you can omit this option.

On Windows you can use [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) or SFTP client like [FileZilla](https://filezilla-project.org/)

### Start CADLAB swarm

Now you're all good to deploy your swarm stack. Depending on whether you want to deploy CADLAB as a stand-alone app or with an external git back-end, you need to execute a `docker stack deploy` command with a specific stack file.

To install a stand-alone version of CADLAB, you need to execute the following command in the `/var/cadlab` directory:

```bash
docker stack deploy --with-registry-auth -c stack.yml cadlab
```

To install CADLAB with an external git back-end, like GitLab, use this command: 

```bash
docker stack deploy --with-registry-auth -c stack-external-git.yml cadlab
```

Starting CADLAB for the first time will take quite some time, as Docker needs to download all the required images and perform the initial installation. 

The stand-alone version of CADLAB is currently running 5 services:
- cadlab
- git-server
- mail-server
- mysql
- nginx

If you run CADLAB with an external git back-end like GitLab, then the `git-server` service is not included, and you should have 4 services running.

### Checking services status

After you started your swarm, you need to make sure that all services started successfully. In order to do it, you can run the following Docker command:

```bash
docker stack ps cadlab
```

It should list all running or failed services in your CADLAB stack.

![docker stack ps](documentation/images/docker-stack-ps.png "Docker stack ps command output")

You can see that some containers are already running, while others are preparing or starting. If everything is going well, after some time, you should see that all containers are running.

You can also list all services by executing the following Docker command:

```bash
docker service ls
```

The output of this command will be similar to this one:

![docker service ls](documentation/images/docker-service-ls.png "Docker service ls command output")

The healthy application state is when you have all services running with `1/1` in the replicas column.

## Troubleshooting

### Checking container health
If not all of your services are running properly, you will see that in the output of the `docker stack ps cadlab` command or `docker service ls` command. Usually, a service won't start because of an incorrect `cadlab.json` file. For example, if you made a mistake in the `automatic_backups` setting, the output of the `ps` command would look like this:

![docker stack ps cadlab - failed service](documentation/images/docker-stack-ps-failed.png "Docker stack ps command output")

On the screenshot above, we see that after the cadlab container failed, Docker tried to restart it 3 more times but couldn't do it. We also see that there was a non-zero exit from the container, so we need to inspect logs to see what went wrong.

If you list all services, you will also notice that cadlab service has `0/0` in the replicas column, which means that the cadlab container is not running:

![docker service ls - failed service](documentation/images/docker-service-ls-failed.png "Docker service ls command output")

To inspect logs for a service, you need to perform the following command:

```bash
docker service logs cadlab_cadlab
```

The last argument in this command is the service name, which you can find from the `docker service ls` command. You can also add the `-f` flag to the logs command to view logs in real-time like so `docker service logs -f cadlab_cadlab` and exit logs by pressing `ctlr+c`.

Below is the log output for the failed cadlab container:

![docker service logs](documentation/images/docker-service-logs.png "Docker service logs command output")

We can see that the `automatic_backups` setting was incorrectly configured in the cadlab.json file.

After fixing the mistake in the cadlab.json file, you need to restart your failed service. First, you need to delete the failed service(s), those with `0/0` replicas, by executing the following command:

```bash
docker service rm cadlab_cadlab
```
- where `cadlab_cadlab` is the name of the service. 

If you have more than one failed service, you can delete all of them individually or delete the whole stack like this:

```bash
docker stack rm cadlab
```

After you fixed the cadlab.json file and removed all failed services, you need to redeploy your stack by executing the command from the [Start CADLAB swarm](#start-cadlab-swarm).

If any container doesn't start, you need to repeat the troubleshooting steps.

### 502 bad gateway
Sometimes you can get a 502 bad gateway error when trying to access the website even though all docker services are running without errors. It usually happens when you try to access the website while CADLAB is still being installed. In this case, wait a minute or two and try refreshing your browser.

## Changing CADLAB configurations

Sometimes you may need to change the CADLAB configuration after the app was successfully deployed. In order to do this, you need to modify the `cadlab.json` file and execute a reconfigure command for CADLAB.

After you modified your `cadlab.json` file, execute the following command:

```bash
docker exec -it cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc cadlab reconfigure
```
**Note**: `cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc` part is a dynamically generated container name in a Docker swarm. But you don't need to type it manually. When you write the `exec` command, just type `docker exec -it cadlab_cadlab` and press `tab`. This will autocomplete the container name. After that, type the `cadlab` utility and CADLAB command to execute, in this case, `reconfigure`.

If autocomplete didn't work on your machine, you could use an alternative way to execute this command. First, list all container by running the following command:

```bash
docker container ls
```

![docker container ls](documentation/images/docker-container-ls.png "Docker container ls command output")

Then copy the container ID of the cadlab container and use this ID instead of the container name like this:

```bash
docker exec -it 6803aaa81bbf cadlab reconfigure
```

## Updating CADLAB

**Important:** Please [make a backup](#backing-up-cadlab) before updating your CADLAB installation. It will help restore your data if the update process goes not according to plan.

When a new release of CADLAB is available, you need to pull changes from this repository if you use git or download a new release from the Releases page to the `/var/cadlab` directory.

Then perform the same `docker stack deploy` command we've already covered in the [Start CADLAB swarm](#start-cadlab-swarm) section.


## Backup & restore CADLAB

### Backing up CADLAB
When you configure CADLAB, you can set the `automatic_backups` option in the `cadlab.json` file to make CADLAB perform backups on a regular basis. Read about this setting [here](#automatic_backups). CADLAB database, files, and git data (for stand-alone installation) will be automatically backed up and stored in the `/var/cadlab/backups` directory. CADLAB will store 10 of the most recent backup files. Backups are named using the following format `[current_timestamp]-yyyy-mm-dd_cadlab.tar.gz`.

You can also perform backups manually using the CADLAB command-line utility. In order to do this, we need to execute a command in the `cadlab` container. 

```bash
docker exec -it cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc cadlab backup
```
**Note**: `cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc` part is a dynamically generated container name in a Docker swarm. See the [Changing CADLAB configurations](#changing-cadlab-configurations) section for guidance on how to find out what is your unique container name.

Depending on your backup strategy, you then can copy backups to Amazon S3 or another server in your network.

### Restoring CADLAB

When you need to restore CADLAB data from a backup, you can do that using the CADLAB command-line utility. By default, CADLAB will restore the most recent backup from the `/var/cadlab/backups` directory. In order to do this, execute the following command:

```bash
docker exec -it cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc cadlab restore
```
**Note**: `cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc` part is a dynamically generated container name in a Docker swarm. See the [Changing CADLAB configurations](#changing-cadlab-configurations) section for guidance on how to find out what is your unique container name.

The command-line utility will tell you what backup it is going to restore and ask for your confirmation.

If you need to restore a specific backup, you can add a `--file=FILENAME` option to specify the filename of the backup you want to restore. Please note that this backup should be in the `/var/cadlab/backups` directory.

```bash
docker exec -it cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc cadlab restore --file=1602549045-2020-10-13_cadlab.tar.gz
```

## CADLAB command-line utility

CADLAB offers a command-line utility to help you manage your CADLAB installation. It currently allows you to clear the application cache, backup, restore, and reconfigure CADLAB. The utility executable is called `cadlab` and can be used from within the `cadlab` container. Run the following Docker command to view command-line utility help output:

```bash
docker exec -it cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc cadlab
```
**Note**: `cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc` part is a dynamically generated container name in a Docker swarm. See the [Changing CADLAB configurations](#changing-cadlab-configurations) section for guidance on how to find out what is your unique container name.

To execute one of the commands, for example, `reconfigure`, run the following Docker command:

```bash
docker exec -it cadlab_cadlab.w1ju1zaqlrpg5rbiqr9engr9n.0foep2va9ua1adzori4kj2ksc cadlab reconfigure
```

## Integrating with GitLab

If you chose to install CADLAB with an external git back-end, for example, GitLab, you need to use the `stack-external-git.yml` file to [start the swarm](#start-cadlab-swarm). After the CADLAB application successfully starts, you need to integrate it with your self-hosted GitLab installation.

Open the URL you specified in the `hostname` [here](#hostname) in your browser, and you should see a CADLAB welcome screen:

![CADLAB.io Welcome screen](documentation/images/cadlab-gitlab-url.png "CADLAB Welcome screen")

You need to enter the URL of your GitLab installation on this screen so that CADLAB can connect to GitLab's API. **Please note**: if your GitLab installation is deployed to a private network, CADLAB also needs to be deployed to the same network so that it's able to connect to GitLab's API.

Submit the form and proceed to the next step:

![CADLAB.io create integration](documentation/images/cadlab-gitlab-callback-url.png "Integration creation step")

If CADLAB was able to connect to your GitLab, you should see the integration settings form like in the screenshot above. Copy the Callback URL from the form and proceed to create an application in GitLab.

In your GitLab, go to Admin area (1), then Applications (2) in the left-hand side navigation, and click "New Application". 

![GitLab New Application](documentation/images/gitlab-new-applicatoin.png "GitLab New Application")

On the "New Application" page, provide application name (1), paste CADLAB callback URL (2), tick "api" checkbox (3), and submit the form.

GitLab will create an application and generate Application ID and Secret key: 

![GitLab Application](documentation/images/gitlab-applicatoin-id.png "GitLab Application")

Copy them over to the corresponding fields in the CADLAB integration form:

![GitLab Application ID and Secret Key](documentation/images/cadlab-gitlab-secret-and-api-key.png "GitLab Application ID and Secret Key")

After clicking the create button, you will be redirected to the sign-in with GitLab screen. 

![GitLab sign in](documentation/images/cadlab-gitlab-sign-in.png "GitLab Application ID and Secret Key")

The first user that signs in during the initial setup will automatically be assigned an organization owner role.