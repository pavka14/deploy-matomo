# Django Deployment Playbooks

This directory contains modular Ansible playbooks for deploying a Django application stack on Ubuntu 24.04. The stack includes:
- PostgreSQL database
- Python (latest) in a virtual environment
- Redis server
- Nginx (with SSL and GeoIP2 integration)
- Uvicorn (for serving Django)
- Automated download and setup of GeoIP2 databases and module
- GitHub repository cloning (using credentials from environment)

## Features
- **Modular design**: Each playbook can be run independently for isolated component deployment or troubleshooting.
- **Environment-driven configuration**: All sensitive and configurable values are loaded from the `.env` file.
- **GeoIP2 support**: Nginx is recompiled with the GeoIP2 module and configured to use MaxMind databases.
- **SSL support**: Automated Let's Encrypt certificate acquisition and renewal.

Note that Nginx gets installed with a config file defined in the templates/nginx.conf.j2 jinja template, which is somewhat simplistic (e.g. has no serving of static files). You will probably want to modify it to suit your particular case.
If you want country and city names in a language other than English, the options to change that are in the geoip2_nginx.conf.j2 template.

## Prerequisites
- Ubuntu 24.04 target server(s) with sudo access
- Ansible 2.18.8 or newer (on host system)
- MaxMind GeoIP2 account and license key
- GitHub access token for private repo cloning (if needed)

### Host System Prerequisites
These steps must be performed on the machine where you run Ansible (the control node):

1. **Install Ansible**
   ```bash
   python3 -m pip install --user ansible
   # Or use your OS package manager (e.g. apt, dnf, brew)
   # Ubuntu example:
   sudo apt update && sudo apt install ansible
   ```

2. **Install the community.postgresql collection**
   ```bash
   ansible-galaxy collection install community.postgresql
   ```
   This is required for database setup tasks in `postgres.yml`.

## Setup Instructions

1. **Copy and edit environment file**
   ```bash
   cp .env.example .env
   # Edit .env with your actual values (domain, passwords, MaxMind credentials, etc.)
   ```

2. **Export environment variables**
   ```bash
   export $(grep -v '^#' .env | xargs)
   ```

3. **Edit inventory**
   - Edit `inventory.ini` to list your target server(s) under the `[django_servers]` group.

## Running the Full Deployment

To run the entire stack in order:
```bash
ansible-playbook -i inventory.ini site.yml
```
This will execute all component playbooks in sequence, deploying the full Django stack.

## Running a Playbook in Isolation

Each playbook is self-contained and can be run independently. For example, to run only the Nginx setup:
```bash
ansible-playbook -i inventory.ini nginx.yml
```
Or to run only the database setup:
```bash
ansible-playbook -i inventory.ini postgres.yml
```

## Validation and Testing

Before running any playbook, always validate syntax and variable resolution:
```bash
export $(grep -v '^#' .env | xargs)
ansible-playbook -i inventory.ini --syntax-check <playbook.yml>
```
For a dry-run (no changes made):
```bash
export $(grep -v '^#' .env | xargs)
ansible-playbook -i inventory.ini --check <playbook.yml>
```

## Playbook List
- `prerequisites.yml` - System packages and fail2ban
- `postgres.yml` - PostgreSQL install and config
- `python.yml` - Python and virtualenv setup
- `redis.yml` - Redis install and config
- `nginx.yml` - Nginx install, SSL, GeoIP2 module compilation, config
- `geoupdate.yml` - GeoIP2 database download, configuration, and scheduled updates
- `django.yml` - Django app deployment, repo clone, migrations, static files, Uvicorn
- `ssl.yml` - Let's Encrypt certificate setup
- `uvicorn.yml` - Uvicorn systemd service setup
- `site.yml` - All of the above in sequence

## Notes
- All configuration values (paths, credentials, repo URLs, etc.) have to be exported from `.env` and are then loaded via `variables.yml`.
- GeoIP2 databases are downloaded using your MaxMind credentials.
- nginx is recompiled with the GeoIP2 module for IP geolocation support.
- nginx.conf is not fully done yet and needs some manual tinkering after the playbook is done. Specifically, one of the problems is that nginx.conf (the main file, not just the server definition) needs this line:
   ```
   load_module modules/ngx_http_geoip2_module.so;
   ```
  but the playbook doesn't add it (yet).
- also, there is a template for the GeoIP2 configuration (geoip2.conf.j2); adding it as a separate .conf file is a wrong approach, it needs to be included in the main nginx.conf file directly.
- the "user" part of the server template is commented out - nginx says it is not allowed thre, this needs to be done correctly.

Playbooks are mostly self-contained, but:
- ssl should run after nginx (confirming domain ownership needs a running web server)
- uvicorn should run after django (the app directory is referenced in the service config, and also a change is made in the django virtual environment)

The Postgres playbook creates an empty database. If you want to move a live site, this is a bit more involved, as you need to:
- do a "rough" dump of the database (e.g. with pg_dump) and move it to the new server
- then get the new server to sync to the old one (run as slave) until they are fully in sync
- move the actual site to the new server
- stop database replication, make the new server standalone (not slave), swap the application server to start using it

The above will involve some downtime, and so will the swap of the web server - in order to get the new SSL certificate, you need the DNS to point to the new server, which doesn't work yet... (and keep in mind that when you change DNS settings, it may take up to 48 hours for them to propagate).
In short, if your app is mission-critical and you cannot afford downtime, you need a lot more than this playbook.

When installing Django, the playbook pulls your "master" branch. If you repository is politically correct, you will need to rename it to "main" in django.yml.

For uvicorn, you can add its own .env file by adding:
EnvironmentFile={{ django_app_dir }}/.env
to the uvicorn.service.j2 template (and then make sure the file is there).
The current setup does not have any environment variables for uvicorn, and supposes that django loads them after it starts with load_env().
You may need uvicorn variables if you want anything to be set before django starts.

## Troubleshooting
- Ensure all required environment variables are set in `.env`.
- Validate each playbook with `--syntax-check` before running.
- Check Ansible output for missing variables or connection errors.

## Short cheatsheet for related commands:
In your old database server:
```bash
SELECT pg_size_pretty(pg_database_size('<your_database_name>'));
```
This will give you the size of the database in a human-readable format.

**From the new server**, to copy the database directly server-to-server over SSH with no intermediate dump file:
```bash
ssh <user>@<old_server> "sudo -u postgres pg_dump -Fc <old_database_name>" | pv | sudo -u postgres pg_restore -d <new_database_name
```
Add "--clean --if-exists" to the pg_restore command above if you want to drop existing tables before restoring (**this will delete data**).

Explanation:
The command above logs into the old server ("ssh"), changes to the database user ("sudo -u postgres"), dumps the database in a custom format ("pg_dump -Fc"), and then pipes it through "pv" (for progress monitoring) to the new server, where it is restored into the new database ("pg_restore -d <new_database_name>").

For that to work, the user running the command on the new server needs to be able to log into the old server without being asked for a password (which will break the "pipe" command to pv); so, save it first:
```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id <user>@<old_server>
```
This will ask you for the password and will save it (on the new server) for future logins without a prompt.

On the remote server, the remote user needs to be able to "su" to postgres without being asked for a password. So, run "sudo visudo" and add the line:
```
<remote_user>  ALL=(postgres) NOPASSWD: /usr/bin/pg_dump
```
Note that after you move the database, its size on the new server will be smaller than what it showed on the old server, because pg_dump only dumps the actual data, and not the "empty" space in the database files (in effect, like "vacuum" and then move).
And the progress bar will show a smaller size still, because it shows the actual transfer size, in which the data is compressed.

---

For more details, see the main repository README and the comments in each playbook.
