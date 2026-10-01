<!--
SPDX-FileCopyrightText: 2026 Bergruebe

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Beszel

This is an [Ansible](https://www.ansible.com/) role which installs the [Beszel](https://beszel.dev/) hub to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Beszel is a lightweight server monitoring platform. It consists of two parts: the **hub**, a web application (built on [PocketBase](https://pocketbase.io/)) which shows historical data, Docker statistics and alerts, and the **agent**, which runs on every system you want to monitor and sends its metrics to the hub.

This role installs the hub only. The agents are to be installed separately on the systems to be monitored.

See the project's [documentation](https://beszel.dev/guide/what-is-beszel) to learn what Beszel does and why it might be useful to you.

## Prerequisites

Beszel needs no separate database server. The hub stores its data (an SQLite database managed by PocketBase, and the SSH key pair used to connect to agents) in the directory specified with `beszel_data_path`.

## Adjusting the playbook configuration

To enable Beszel with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# beszel                                                               #
#                                                                      #
########################################################################

beszel_enabled: true

########################################################################
#                                                                      #
# /beszel                                                              #
#                                                                      #
########################################################################
```

### Set the hostname

To enable the Beszel instance you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
beszel_hostname: "beszel.example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

### Set the path prefix (optional)

Beszel can be hosted under a subpath. To do so, add the following configuration to your `vars.yml` file:

```yaml
# The value must either be `/` or not end with a slash.
beszel_path_prefix: /beszel
```

The public URL of the hub (Beszel's `APP_URL`), which is used for links in emails and notifications, is derived from `beszel_hostname` and `beszel_path_prefix` automatically. You can override it with `beszel_app_url`.

### Create the initial user (optional)

By default, Beszel asks you to create the first user account on the web interface. You can also have the account created automatically on the first start by adding the following configuration to your `vars.yml` file:

```yaml
beszel_environment_variable_user_email: admin@example.com
beszel_environment_variable_user_password: YOUR_PASSWORD_HERE
```

>[!WARNING]
> These values only take effect on the very first start of the hub. Changing them afterwards has no effect — change the account on the web interface instead.

### Connect a local agent via Unix socket (optional)

If you run a Beszel agent on the same host as the hub, the hub can connect to it via a Unix socket instead of SSH over the network. To enable it, add the following configuration to your `vars.yml` file:

```yaml
beszel_agent_socket_enabled: true
```

With this setting, the directory specified with `beszel_agent_socket_path_host` (by default `beszel_base_path`/socket) is mounted into the hub container at `/beszel_socket`.

Configure the agent to listen on a socket in this directory (e.g. with `LISTEN=/beszel/socket/beszel.sock` on the host, adjusted to your `beszel_base_path`), and add the system on the hub's web interface with `/beszel_socket/beszel.sock` as its host.

>[!NOTE]
> The hub container runs as the user specified with `beszel_uid` and `beszel_gid`, not as root. Connecting to a Unix socket requires write permission on the socket file, and the agent does not change the permissions of the socket it creates. Make sure that the agent runs as the same user (or group, with a umask which keeps the socket group-writable) as the hub. Otherwise the hub fails to connect to the agent with a "permission denied" error.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `beszel_environment_variables_additional_variables` variable

See [this page](https://beszel.dev/guide/environment-variables) of the official documentation for Beszel's config options that you could put in `beszel_environment_variables_additional_variables`.

## Hardening the container

The hub container is started with the following hardening options:

- it runs as the non-root user specified with `beszel_uid` and `beszel_gid`
- all Linux capabilities are dropped (`--cap-drop=ALL`) and privilege escalation is prevented (`--security-opt=no-new-privileges`)
- the root filesystem is read-only (`--read-only`)

The only writable locations inside the container are the data directory (`/beszel_data`, bind-mounted from `beszel_data_path`), the optional agent socket directory (`/beszel_socket`), and `/tmp`, which is a `tmpfs` mount whose size is controlled by `beszel_container_tmpfs_tmp_size`.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, the Beszel hub becomes available at the specified hostname like `https://beszel.example.com`.

To get started, open the URL with a web browser and create the first user account (unless you have it created automatically). Then add the systems to be monitored by clicking "Add System". The dialog shows the public key and the token required for installing an agent on each system.

Agents can connect to the hub in two ways:

- **The hub connects to the agent via SSH**: the agent listens on a port (`45876` by default) on the monitored system, which needs to be reachable from the hub container.
- **The agent connects to the hub via WebSocket**: the agent is configured with the hub's URL (`HUB_URL`) and a token. This only requires the hub to be reachable via HTTPS, which the reverse proxy takes care of.

See [this page](https://beszel.dev/guide/agent-installation) on the official documentation for details about installing agents.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu beszel` (or how you/your playbook named the service, e.g. `mash-beszel`).
