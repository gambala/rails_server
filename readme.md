# What is this?

This is an automated script to help you setup new server and deploy Rails application along with database dump.

## Requirements:
Ubuntu 16.04 or newer

## Setup

    cd your-rails-project-folder
    git clone git@github.com:gambala/rails_server.git server
    cp server/hosts.example.yml server/hosts.yml
    ansible-galaxy install -r server/requirements.yml

Also:

1. Place your postgres database dump at `server/app_name.sql`. Dump must be created with following command:

`pg_dump --clean --format c --verbose --blobs --file database_name.dump database_name`
`scp -P your_port your_user@your_host:database_name.dump server/files`
`scp -P 4000 deploy@site.com:database_name.dump server/files`

`tar -czvf app_name_staging_uploads.tar.gz apps/app_name/shared/public/uploads/`
`scp -P 4000 deploy@site.com:app_name_staging_uploads.tar.gz server/files`

2. Update IP address of your server in `config/deploy/production.rb` and set user value to `deploy`.

**IMPORTANT**
After first deploy and restore database from backup **don't forget** to change `restore_backup` value to `false`. In other case on next applying `app.yml` playbook your database will be rewrited with provided in `server` directory.

## Usage

Run
```
ansible-playbook server/setup-initial.yml -i server/hosts.shared.yml -i server/hosts.production.yml -e 'ansible_port=22'
ansible-playbook server/setup-server.yml -i server/hosts.shared.yml -i server/hosts.production.yml
cap production deploy:check
ansible-playbook server/setup-app.yml -i server/hosts.shared.yml -i server/hosts.production.yml
ansible-playbook server/restore-data.yml -i server/hosts.shared.yml -i server/hosts.production.yml
```

This script goes through full server configuration process. For now it does next things (in order of applying):

### setup-initial.yml

- Installs Python2 in order to allow Ansible do all the work
- Installs pip, python packet manager
- Creates `deploy` user which is our main user for application deployments
- Configures `zsh` and `oh-my-zsh` for `root` user
- Installs `vim` and `htop`

### setup-server.yml

- Configures `zsh` and `oh-my-zsh` for `deploy` user
- Installs `monit` for server state monitoring and notifications (RAM, HDD, etc)
- Installs `nginx` with `passenger` to serve your Rails applications
- Installs `Redis`
- Installs `Ruby`
- Installs `PostgreSQL`

### setup-app.yml

- Configures Nginx to server your Rails application
- Creates PostgreSQL database and user for your Rails application
- Uploads database dump and restores it on server
- Setups daily DB backups with uploading to Yandex Disk

### Application

Run `cap production deploy` to start deployment process.

### That's all!

## Deploying application to already configured server

Just run `ansible-playbook server/setup-app.yml -i server/hosts`. You'll also want to update files from Preparation as appropriate.
