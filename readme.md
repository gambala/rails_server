<img src="https://github.com/gambala/rails_server/raw/master/logo.png" alt="rails_server logo" width="256" height="256">

# What is it?

`rails_server` is a bunch of Ansible playbooks, that helps you to setup new server and to deploy Rails application along with database dump.

## What technologies it covers?

- Ubuntu 16.04 or newer
- ZSH, Oh My ZSH!
- PostgreSQL
- NodeJS
- ImageMagick
- Yarn
- Rust
- Redis
- RVM, Ruby MRI
- Monit
- Let's Encrypt (certbot)

Also there are some hardcoded limits:

- Ubuntu and PostgreSQL user can be named only as `deploy`

## Setup

    cd your-rails-project-folder
    git clone git@github.com:gambala/rails_server.git server
    echo /server >> .gitignore
    git commit -am "Gitignore server folder"
    cp server/hosts.all.example.yml server/hosts.all.yml
    cp server/hosts.production.example.yml server/hosts.production.yml
    ansible-galaxy install -r server/requirements.yml

## Usage

### With brand new server

1. Create server on your hosting
2. Update credentials in server/hosts files and in capistrano config files
3. Run commands:

```
ansible-playbook server/setup-initial.yml -i server/hosts.all.yml -i server/hosts.production.yml -e 'ansible_port=22'
ansible-playbook server/setup-server.yml -i server/hosts.all.yml -i server/hosts.production.yml
```

### With configured server

1. Run commands:

```
ansible-playbook server/setup-app.yml -i server/hosts.all.yml -i server/hosts.production.yml
cap production setup
cap production puma:nginx_config
cap production deploy
```

2. Ssh to your new server and run command:

```
sudo service nginx restart
```

or for new server:

```
sudo service nginx start
```

### With data from old server

1. Ssh to your old server
2. Make DB dump and uploads (from carrierwave for example) archive:

```
pg_dump --clean --format c --verbose --blobs --no-owner --file project_production.dump project_production
tar -czvf project_production_uploads.tar.gz apps/app_name/shared/public/uploads/
```

3. Download these files to your `server/files` folder:

```
scp -P 4000 deploy@site.com:project_production.dump server/files
scp -P 4000 deploy@site.com:project_production_uploads.tar.gz server/files
```

4. Run role:

```
ansible-playbook server/restore-data.yml -i server/hosts.all.yml -i server/hosts.production.yml
```

## Under the hood

Each playbook runs specific roles, listed below.

### setup-initial.yml

- Installs Python2 in order to allow Ansible do all the work
- Installs pip, python packet manager
- Creates `deploy` user which is our main user for application deployments
- Configures `git`, `zsh` and `oh-my-zsh` for `root` user

### setup-server.yml

- Configures `zsh` and `oh-my-zsh` for `deploy` user
- Adds `en_US.UTF-8` locale
- Installs `curl`, `htop`, `landscape-common`, `ncdu` and `vim`
- Installs and configures `fail2ban`
- Installs NodeJS
- Adds 2GB swapfile
- Installs Nginx with specific configs (one of them includes the code for certbot)
- Installs ImageMagick, Yarn and Rust
- Installs Redis
- Installs RVM and Ruby
- Installs PostgreSQL and creates `deploy` superuser in it
- Installs Monit for server state monitoring and notifications (RAM, HDD, etc)
- Installs Certbot and registers in it with email from hosts

### setup-app.yml

- Creates PostgreSQL database for your Rails application
- Issues ssl certificates with certbot for domains from your hosts
- Automates daily DB backups with uploading to AWS S3

### restore-data.yml

- Uploads database dump and restores it on server
- Uploads `uploads` archive and unzip in on server

# Additional steps

## Turn monit web interface on

Run `sudo nano /etc/monit/monitrc` and write this to file:

```
set daemon 30

set logfile /var/log/monit.log

set idfile /var/lib/monit/id

set statefile /var/lib/monit/state

set eventqueue
    basedir /var/lib/monit/events # set the base directory where events will be stored
    slots 100 # optionally limit the queue size

set httpd port 2812 and
    allow admin:monit      # require user 'admin' with password 'monit'

include /etc/monit/conf.d/*
include /etc/monit/conf-enabled/*
```

Then run `sudo monit reload`.

## Add nginx to monit

Run `sudo nano /etc/monit/conf-enabled/nginx` and write this:

```
 check process nginx with pidfile /var/run/nginx.pid
   group www
   group nginx
   start program = "/etc/init.d/nginx start"
   stop program = "/etc/init.d/nginx stop"
#  if failed port 80 protocol http request "/" then restart
   if 5 restarts with 5 cycles then timeout
   depend nginx_bin
   depend nginx_rc

 check file nginx_bin with path /usr/sbin/nginx
   group nginx
   include /etc/monit/templates/rootbin

 check file nginx_rc with path /etc/init.d/nginx
   group nginx
   include /etc/monit/templates/rootbin
```

Then run `sudo monit reload`.
