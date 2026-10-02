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

### Single sign-on with OIDC, e.g. Pocket ID (optional)

Beszel supports logging in with an OAuth2/OIDC provider such as [Pocket ID](https://pocket-id.org/). The provider itself is configured on Beszel's PocketBase admin interface (it cannot be set with environment variables), while the login behavior is controlled with the following variables:

```yaml
# Create a Beszel user automatically on the first OIDC login.
# Without it, a user with the same email address has to exist in Beszel already.
beszel_environment_variable_user_creation: true

# Open the login page of the provider in the same window instead of a popup.
beszel_environment_variable_oauth_disable_popup: true

# Optional: allow logging in with OIDC only.
# Enable it only after having confirmed that the OIDC login works.
beszel_environment_variable_disable_password_auth: true
```

>[!NOTE]
> These settings are enforced on every start of the hub. Change them with the variables above, not on the PocketBase admin interface, as changes made there are overwritten.

#### Communication between the hub and the provider

During the login, the browser is redirected to the provider's public URL, while the hub itself has to request a token and the user info from the provider. The hub can do the latter over a container network, without leaving the server. Pocket ID supports this setup out of the box: the authorization endpoint is served on its public URL, while the token and user info endpoints can be requested on its internal address.

For this, the hub container needs to be connected to the provider's container network:

```yaml
# The container network of Pocket ID (`pocket_id_container_network`)
beszel_container_additional_networks_custom:
  - pocket_id
```

>[!NOTE]
> If you use the [MASH playbook](https://github.com/mother-of-all-self-hosting/mash-playbook), the hub is connected to the network of the Pocket ID instance installed by the playbook automatically.

#### Creating the OIDC client on Pocket ID

On Pocket ID's admin interface, go to "OIDC Clients" and add a client with the following callback URL:

```txt
https://beszel.example.com/api/oauth2-redirect
```

If the hub is reachable on several URLs (e.g. via Traefik and via [Tailscale](#using-beszel-with-tailscale-tsdproxy-optional)), add a callback URL for each of them. Then note down the client ID and the client secret.

#### Adding the provider on Beszel

Log in to the PocketBase admin interface at `https://beszel.example.com/_/` with a superuser account (the first user created on Beszel is a superuser as well), and then:

1. Open "Settings", and turn off "Hide collection create and edit controls".
2. Go to "Collections", open the settings of the `users` collection, and select the "Options" tab.
3. Enable "OAuth2", click "Add provider", and select "OpenID Connect".
4. Fill in the fields as below, replacing `pocketid.example.com` with Pocket ID's hostname and `mash-pocket-id` with its container name (`pocket_id_identifier`):

   | Field | Value |
   | --- | --- |
   | Client ID / Client secret | the values from Pocket ID |
   | Display name | `Pocket ID` |
   | Auth URL | `https://pocketid.example.com/authorize` |
   | Token URL | `http://mash-pocket-id:1411/api/oidc/token` |
   | Fetch user info from | User info URL |
   | User info URL | `http://mash-pocket-id:1411/api/oidc/userinfo` |
   | Support PKCE | enabled |

Only the auth URL is opened by the browser, so it has to be the public one. The token and user info URLs are requested by the hub over the container network, and therefore point to Pocket ID's container and its port (`pocket_id_container_http_port`, `1411` by default).

After saving the settings, the login page of Beszel shows a button to log in with Pocket ID.

### Using Beszel with Tailscale (TSDProxy) (optional)

[TSDProxy](https://almeidapaulopt.github.io/tsdproxy/) makes containers reachable in your Tailscale network (tailnet) at `https://<name>.<tailnet>.ts.net`, with a TLS certificate issued by Tailscale. This is useful for Beszel in two ways: agents can connect to the hub over the tailnet, and the web interface can optionally be made reachable over the tailnet only.

#### Exposing the hub via TSDProxy

To have TSDProxy expose the hub, add the following configuration to your `vars.yml` file:

```yaml
beszel_container_labels_tsdproxy_enabled: true

# The machine name in the tailnet. Defaults to the container name (`beszel_identifier`).
# beszel_container_labels_tsdproxy_name: beszel
```

TSDProxy needs to share a container network with the hub. If you install TSDProxy yourself, add its network to `beszel_container_additional_networks_custom` (or the other way round). If you use the [MASH playbook](https://github.com/mother-of-all-self-hosting/mash-playbook), this is done automatically when its TSDProxy service is enabled.

After installing, the hub becomes available at `https://<name>.<tailnet>.ts.net` (e.g. `https://mash-beszel.tail1234.ts.net`) for every device in your tailnet. TSDProxy forwards WebSocket connections, which the agents need.

>[!NOTE]
> TSDProxy serves the hub at the root of the tailnet hostname, so this cannot be combined with hosting Beszel under a subpath (`beszel_path_prefix`).

#### Connecting agents over Tailscale

The recommended way of connecting agents over the tailnet is to have **the agent connect to the hub** via WebSocket. With it, the hub does not need to reach the systems to be monitored, and the agents do not need to open a port. This matters, as the hub container itself is not a member of the tailnet: it cannot connect to agents at their `100.x.y.z` or `*.ts.net` addresses.

1. Make sure the system to be monitored is a member of the tailnet, i.e. it runs Tailscale and can open `https://<name>.<tailnet>.ts.net`.
2. On the hub's web interface, click "Add System" and copy the `KEY` and `TOKEN` values from the agent configuration shown there.
3. Install the agent on the system (see [the official documentation](https://beszel.dev/guide/agent-installation)), and set `HUB_URL` to the tailnet URL of the hub:

   ```yaml
   services:
     beszel-agent:
       image: henrygd/beszel-agent
       container_name: beszel-agent
       restart: unless-stopped
       network_mode: host
       volumes:
         - /var/run/docker.sock:/var/run/docker.sock:ro
       environment:
         HUB_URL: https://mash-beszel.tail1234.ts.net
         TOKEN: YOUR_TOKEN_HERE
         KEY: "ssh-ed25519 YOUR_PUBLIC_KEY_HERE"
   ```

The agent opens the connection to the hub through the tailnet, and the hub sends its requests back over the same WebSocket connection.

>[!TIP]
> On the server where the hub runs, an agent can be connected via a [Unix socket](#connect-a-local-agent-via-unix-socket-optional) instead.

#### Making the web interface reachable over Tailscale only

To have the web interface reachable only from devices in your tailnet, disable the Traefik labels and set the tailnet hostname as `beszel_hostname`, so that `APP_URL` (used for links in notifications) points to it:

```yaml
beszel_container_labels_traefik_enabled: false

beszel_container_labels_tsdproxy_enabled: true

# <name>.<tailnet>.ts.net, where <name> is `beszel_container_labels_tsdproxy_name`
beszel_hostname: mash-beszel.tail1234.ts.net
```

The hub is then not reachable from the internet at all, and agents can only connect to it over the tailnet as described above.

Single sign-on with Pocket ID keeps working in this setup, as long as Pocket ID itself is reachable for your browser: the browser opens Pocket ID's auth URL, while the hub talks to Pocket ID over the container network. Make sure to register `https://mash-beszel.tail1234.ts.net/api/oauth2-redirect` as the callback URL on Pocket ID.

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
