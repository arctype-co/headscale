# Running headscale on Linux

## Goal

This documentation has the goal of showing a user how-to set up and run `headscale` on Linux.
In additional to the "get up and running section", there is an optional [SystemD section](#running-headscale-in-the-background-with-systemd)
describing how to make `headscale` run properly in a server environment.

## Configure and run `headscale`

1. Download the latest [`headscale` binary from GitHub's release page](https://github.com/juanfont/headscale/releases):

```shell
wget --output-document=/usr/local/bin/headscale \
   https://github.com/juanfont/headscale/releases/download/v<HEADSCALE VERSION>/headscale_<HEADSCALE VERSION>_linux_<ARCH>
```

2. Make `headscale` executable:

```shell
chmod +x /usr/local/bin/headscale
```

3. Prepare a directory to hold `headscale` configuration and the [SQLite](https://www.sqlite.org/) database:

```shell
# Directory for configuration

mkdir -p /etc/headscale

# Directory for Database, and other variable data (like certificates)
mkdir -p /var/lib/headscale
```

4. Create an empty SQLite database:

```shell
touch /var/lib/headscale/db.sqlite
```

5. Create a `headscale` configuration:

```shell
touch /etc/headscale/config.yaml
```

It is **strongly recommended** to copy and modifiy the [example configuration](../config-example.yaml)
from the [headscale repository](../)

6. Start the headscale server:

```shell
  headscale serve
```

This command will start `headscale` in the current terminal session.

---

To continue the tutorial, open a new terminal and let it run in the background.
Alternatively use terminal emulators like [tmux](https://github.com/tmux/tmux) or [screen](https://www.gnu.org/software/screen/).

To run `headscale` in the background, please follow the steps in the [SystemD section](#running-headscale-in-the-background-with-systemd) before continuing.

7. Verify `headscale` is running:

Verify `headscale` is available:

```shell
curl http://127.0.0.1:8080/metrics
```

8. Create a namespace ([tailnet](https://tailscale.com/kb/1136/tailnet/)):

```shell
headscale namespaces create myfirstnamespace
```

### Register a machine (normal login)

On a client machine, execute the `tailscale` login command:

```shell
tailscale up --login-server YOUR_HEADSCALE_URL
```

Register the machine:

```shell
headscale --namespace myfirstnamespace nodes register --key <YOU_+MACHINE_KEY>
```

### Register machine using a pre authenticated key

Generate a key using the command line:

```shell
headscale --namespace myfirstnamespace preauthkeys create --reusable --expiration 24h
```

This will return a pre-authenticated key that can be used to connect a node to `headscale` during the `tailscale` command:

```shell
tailscale up --login-server <YOUR_HEADSCALE_URL> --authkey <YOUR_AUTH_KEY>
```

## Running `headscale` in the background with SystemD

This section demonstrates how to run `headscale` as a service in the background with [SystemD](https://www.freedesktop.org/wiki/Software/systemd/).
This should work on most modern Linux distributions.

1. Create a SystemD service configuration at `/etc/systemd/system/headscale.service` containing:

```systemd
[Unit]
Description=headscale controller
After=syslog.target
After=network.target

[Service]
Type=simple
User=headscale
Group=headscale
ExecStart=/usr/local/bin/headscale serve
Restart=always
RestartSec=5

# Optional security enhancements
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/headscale /var/run/headscale
AmbientCapabilities=CAP_NET_BIND_SERVICE
RuntimeDirectory=headscale

[Install]
WantedBy=multi-user.target
```

2. In `/etc/headscale/config.yaml`, override the default `headscale` unix socket with a SystemD friendly path:

```yaml
unix_socket: /var/run/headscale/headscale.sock
```

3. Reload SystemD to load the new configuration file:

```shell
systemctl daemon-reload
```

4. Enable and start the new `headscale` service:

```shell
systemctl enable headscale
systemctl start headscale
```

5. Verify the headscale service:

```shell
systemctl status headscale
```

Verify `headscale` is available:

```shell
curl http://127.0.0.1:8080/metrics
```

`headscale` will now run in the background and start at boot.
