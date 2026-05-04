# Ansible Playbooks Collection

This repository contains a collection of Ansible playbooks designed for the hobbyist webmaster who needs things to be installed in a rational and easy manner but does not need hard-core "production grade" automation yet.

## Available Playbooks

### [Deploy Matomo](ansible/deploy-matomo/)
This playbook creates a clean Matomo installation with phpMyAdmin on Ubuntu servers. It handles everything from system prerequisites to SSL certificates, resulting in a fully functional web analytics setup.

**Use this when:** You want to set up a fresh Matomo installation from scratch.

### [Replicate MariaDB](ansible/replicate-mariadb/)
This playbook automates the migration of MariaDB databases between servers with replication setup. While originally designed to complement the Matomo deployment for moving existing installations, it can be used to migrate any MariaDB database.

**Use this when:** You need to move an existing database to a new server, or want to set up database replication between servers.

### [Deploy Django](ansible/deploy-django/)
This playbook creates a Django installation (with code pulled from a github repository) with a PostgreSQL database on Ubuntu servers. It handles everything from system prerequisites to SSL certificates - but the database is "clean" (migrations only).

**Use this when:** You want to set up a fresh Matomo installation from scratch.

## Getting Started

1. **For a fresh Matomo installation:** Start with the [Deploy Matomo](ansible/deploy-matomo/) playbook
2. **To migrate an existing Matomo:** First run the Deploy Matomo playbook, then use [Replicate MariaDB](ansible/replicate-mariadb/) to move your data
3**For a fresh Django installation:** Start with the [Deploy Django](ansible/deploy-django/) playbook

## Target Audience

These playbooks are designed for hobbyist webmasters and small-scale deployments. They provide:
- Straightforward, easy-to-understand automation
- Reasonable security measures without over-complexity  
- "Install once" approach rather than enterprise-scale orchestration
- Clear documentation and examples

## Known Issues

- it seems that the SSL installation in deploy-matomo installs certbot and creates a certificate, but this does not renew on expiry; this needs to be investigated and fixed.
- additionally to the above, when the certificate does update, nginx needs to be reloaded by the cron job, otherwise it still keeps using the old certificate.

## Future Plans

More playbooks will be added over time to cover additional common web hosting scenarios and applications.

## Contributing

Submissions of new playbooks and corrections/expansions of existing ones are welcome! Please feel free to:
- Submit pull requests with improvements
- Open issues for bug reports or feature requests
- Share your experiences and suggestions

## Repository Structure

```
ansible/
├── deploy-matomo/     # Matomo deployment playbook
└── replicate-mariadb/ # MariaDB replication and migration
├── deploy-django/      # Django deployment playbook
```
