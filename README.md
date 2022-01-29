# ApisCP backups: Duplicity and Rclone

This playbook is meant to run on a controller node (can be your own pc) in order to perform an automated backups configuration on your ApisCP nodes.

The playbook will configure **Duplicity** for the backups themselves, **Rclone** for remote backup transmission, **systemd** for the required services, timers and reboot guards.

Read more about:
- [Caveats](#caveats)
- [Configuration](#configuration)
- [Usage](#usage)
- [How it works](#how-it-works)
- [Heartbeats](#heartbeats)

## Caveats

Rclone configuration requires you to initially use `rclone` to configure it on the controller node ([Learn how &rarr;](https://rclone.org/docs/#configure)). Then, you'll have to tell the playbook where to find the configuration and it's name, through the `config.yml` similar to what follows:

```yaml
---
# where your rclone config is
rclone_config_source: '/home/yourname/.config/rclone/rclone.conf'
# the name of the remote connection you configured
rclone_config_section: 'google-backups'
```

## Configuration

Create a `config.yml` file containing a similar configuration as described below, omit the defaults:

```yaml
---
# where to store backups
backup_dir: '/home/backup'
# which cipher to use
backup_algo: 'AES256'
# which paths to back up
backup_directories: [
  '/home/virtual/site*/shadow',
  '/home/virtual/site*/info',
  '/var/log/mailer_table*',
  '/var/lib/mysql/mysql-grants*',
  '/var/lib/pgsql/*/backups'
]
# URL to be called on success
heartbeat_url: "https://betteruptime.com/api/v1/heartbeat/{{ heartbeat_id }}"
# your encryption password
backup_secret_passphrase: 'your-secure-password'
# where your rclone config is
rclone_config_source: '/home/yourname/.config/rclone/rclone.conf'
# the name of the remote connection you configured
rclone_config_section: 'google-backups'
```

Create an `inventory.yml` file containing the information of your ApisCP nodes ([Learn how &rarr;](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)) as described below:

```yaml
---
all:
  hosts:
    web1:
      # overriding global configuration
      rclone_config_section: 'eu-backups'
    web2:
      rclone_config_section: 'na-backups'
```

## Usage

1. [Install Ansible](http://docs.ansible.com/intro_installation.html).
   ```shell script
   sudo yum install ansible
   ```
2. Clone this repository to your local drive.
   ```shell script
   git clone https://github.com/thundersquared/apiscp-duplicity-rclone.git ; cd apiscp-duplicity-rclone/
   ```
3. Run playbook.
   ```shell script
   ansible-playbook main.yml
   ```

## How it works

This playbook creates installs `duplicity` and `rclone` on the servers in your inventory. It creates a `systemd` timer to invoke the related service that runs `/usr/local/sbin/SystemBackup` daily. It also installs **a reboot guard**, a unit that prevents system reboots during an ongoing backup.

When a backup starts, the reboot guard is enabled. Duplicity will be using the configuration name provided for rclone to mount and send the backups to the remote connection. When the backup ends, `curl` is used to call the `heartbeat_url` when set then the reboot guard is disabled.

## Heartbeats

Heartbeats are used to notify backup job completion. You can add a `heartbeat_url` to your `config.yml` with a per-node unique ID such as `heartbeat_id` in your `inventory.yml`. Here's an example:

```yaml
# config.yml
---
# where your rclone config is
rclone_config_source: '/home/yourname/.config/rclone/rclone.conf'
# the name of the remote connection you configured
rclone_config_section: 'google-backups'
# URL to be called on success
heartbeat_url: "https://betteruptime.com/api/v1/heartbeat/{{ heartbeat_id }}"
```

```yaml
# inventory.yml
---
all:
  hosts:
    web1:
      heartbeat_id: 'web1-token-123456'
    web2:
      heartbeat_id: 'web2-token-123456'
```

The URLs that the script will be calling will then look something like:

```
# web1 will call:
https://betteruptime.com/api/v1/heartbeat/web1-token-123456
# web2 will call:
https://betteruptime.com/api/v1/heartbeat/web2-token-123456
```
