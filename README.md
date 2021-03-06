# Ansible Role: rsnapshot Backups

# Description

This role can configure both the rsnapshot server and backup clients. It can
also configure an intermediate SSH proxy host that allows server -> proxy ->
client connections for backups.

Terminology: the "server" is the machine that will use rsync to pull the
backups from the "clients". A "proxy" is an intermediate host between the two
endpoints.

# Overview

On the rsnapshot server, the role will create:

  - a backup directory hierarchy, e.g. at **/backup**
  - a directory at **/var/log/rsnapshot** for the backup logs
  - a cron job to schedule backups at **/etc/cron.d/rsnapshot**
  - a backup configuration directory at **/etc/rsnapshot.d**

Each client will have its own rsnapshot.conf file in the **/etc/rsnapshot.d**
directory. This set up allows for:

  - easy customization of rsnapshot options for individual clients
  - simple testing and running of backups for a single client
  - multiple rsnapshot jobs running in parallel depending on network/disk bandwidth

On the rsnapshot clients, the role will create:

  - an 'rsnapshot' user with sudo privileges to run an rsync wrapper script

If an intermediate SSH proxy host is configured for the role, it will also create:

  - an 'sshproxy' account on the proxy host
  - a **~sshproxy/.ssh/authorized_keys** file that limits connections from a single IP
    address and forces passthrough connections
  - an SSH configuration file on the server at **/root/.ssh/config** that
    specifies the path through the proxy to the client

# Required Role Variables

### Server Configuration

Enable the server role in the host_vars file by specifying the filesystem path
where backups will be stored:

```
rsnapshot_backup_root: /backup
```

### Client Variables

Configure client variables in either the host_vars or group_vars file depending on the
setting scope.

Your hosts should have cleanly configured host and domain names as the
rsnapshot configuration files will specify client hostnames as fully qualified
domain names. E.g. in the format {{ ansible_hostname}}.{{ ansible_domain }}.

You must specify the rsnapshot backup server and its IP address in a group or
host variable file:

```
backup_server: the-ansible-inventory-hostname
backup_server_ip: ip.add.re.ss  # This will be the proxy server IP when using one
```

You should specify the client paths to be backed up in an appropriate group or
host variable file:

```
backup_dirs:
  - /etc
  - /root
  - /opt
```

These are the only required settings to backup any number of clients to a single server.

# Optional Role Variables

### Client Configuration

There are a number of tunable configuration options for the rsnapshot.conf files
for each client. See **[defaults/main.yml](defaults/main.yml)** for a complete list.

### Proxy Configuration

The SSH proxy configuration uses a custom SSH client configuration on the
rsnapshot server to build the tunnels used for the rsync backups. The role will
assemble SSH configuration snippets in **/root/.ssh/config.d/** into a single SSH
client configuration file at **/root/.ssh/config**.

To use the proxy functionality, add a list variable of the hosts that the proxy
host will proxy connections for:

```
backup_proxy:
  - hostname1
  - hostname2
```

On the backup client, you should specify the IP address of the proxy for its backup_server_ip
setting:


```
backup_server_ip: proxy-server-IP
```

**Nota Bene:** You will need to connect manually from the server to the proxied
rsnapshot client one time to accept the client SSH host key. If you figure out
a good way to automate this, please open a PR!

# Testing and Troubleshooting Your Configuration

It's a good idea to run your first backup manually. This helps to ensure that
SSH user and host keys are in working order and any tunnelled connections are being
set up properly.

To test the configuration and do a dry run of the backup:

```
rsnapshot -tvc /etc/rsnapshot.d/HOSTNAME.conf sync
```

If it complains about an error in the configuration file, check the file syntax:

```
rsnapshot -c /etc/rsnapshot.d/HOSTNAME.conf configtest
```

If the check passes, run the first backup manually in a screen(1) or tmux(1)
session:

```
rsnapshot -c /etc/rsnapshot.d/HOSTNAME.conf sync
```

Monitor the progress to ensure it completes successfully. Backups run from cron
will log their output to **/var/log/rsnapshot/HOSTNAME.log**.
