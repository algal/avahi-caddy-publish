# Avahi mDNS Publisher

This script solves a particular problem -- publishing multiple mDNS names, which map to the same IP host, optionally sourcing those names from a Caddyfile.

Here is how to run it directly:

``` sh
# Using Caddyfile (automatically extracts .local hostnames)
./avahi-caddy-publish --caddyfile /etc/caddy/Caddyfile --caddy-ip 192.168.1.100

# Using a traditional hosts file
./avahi-caddy-publish -f /etc/avahi/hosts

# Using command line arguments
./avahi-caddy-publish -a 192.168.1.10 foo.local bar.local
```

Here is how to run it as a systemd service, reading from a Caddyfile:

```conf
[Unit]
Description=Avahi publish Caddy .local names
After=avahi-daemon.service
Requires=avahi-daemon.service

[Service]
ExecStart=/usr/local/bin/avahi-caddy-publish --caddyfile /etc/caddy/Caddyfile --caddy-ip 192.168.1.100
Restart=on-failure
ExecReload=/bin/kill -HUP $MAINPID
User=avahi
Group=avahi

[Install]
WantedBy=multi-user.target
```

## Motivation

Do you need this? What does the above mean? Here is the concrete problem and some background.

Suppose that you have a home linux server `box` which runs services on various ports. You want devices on your home network to be able to browse to those services by using clean DNS names like `jupyter.box`. A natural solution for this is Caddy, which provides _virtual hosting_. That is, it configures your server to know those names, in order to map incoming HTTP requests from names to different internal port numbers.

But suppose that, in addition, you don't want to reconfigure every single device on your network, or your router, or a public DNS server, to know about those names. In this case, a natural solution is to use multicast DNS (mDNS) names, like `jupyter.box.local` and `openwebui.box.local`. Multicast DNS lets the server advertise the names itself.

So now your Caddyfile file looks like the following:

```
jupyter.box.local:80 {
	reverse_proxy 127.0.0.1:8888
}

solveit.box.local:80 {
	reverse_proxy 127.0.0.1:5001
}
```

But caddy only configures your server to _respond_ to those names. Your server still needs to _publish_ those mDNS names. This is what avahi is for.

Unfortunately, `avahi-publish --address --no-reverse` only publishes one name at at a time, so you need to create one avahi service definition per name. Also, each service definition is merely duplicating a name from your Caddyfile. Also, the `--no-reverse` flag is critical but rather obscure.

This script solves those problems. It lets you create one systemd service definition that handles publishing mDNS names for all your services. It works by supervising multiple processes running `avahi-publish --address --no-reverse`, and by parsing your Caddyfile for the names. It is designed to run under systemd. It terminates, reloads, and logs output cleanly.

## Installation

To install this so that it publishes mDNS records based on your Caddyfile, follow these steps:

1.  **Copy Files**

    Copy the script and systemd service definition into place:
    ```sh
    $ sudo cp avahi-caddy-publish /usr/local/bin/
    $ sudo cp avahi-caddy-publish.service /etc/systemd/system/
    ```

2.  **Configure the Service**

    Edit the service file to match your environment. You must set the correct IP address for your server and verify the path to your Caddyfile.
    ```sh
    $ sudo systemctl edit --full avahi-caddy-publish.service
    ```
    Inside the editor, find and change the `ExecStart` line:
    ```ini
    ExecStart=/usr/local/bin/avahi-caddy-publish --caddyfile /etc/caddy/Caddyfile --caddy-ip YOUR_SERVER_IP
    ```
    Replace `YOUR_SERVER_IP` with your server's actual IP address (e.g., `192.168.1.100`).

3.  **Set Permissions**

    The service runs as the `avahi` user, which needs permission to read your Caddyfile. You may need to adjust your Caddyfile's permissions to make it readable by the `avahi` user or group.

4.  **Start the Service**

    Finally, load, enable, and start the new service:
    ```sh
    $ sudo systemctl daemon-reload
    $ sudo systemctl enable avahi-caddy-publish
    $ sudo systemctl start avahi-caddy-publish
    ```

### Requirements

This setup requires that you already have caddy, avahi-utils, and Python installed, which you can do with `sudo apt install caddy avahi-daemon avahi-utils python3`.

I use this on Debian 12 (Bookworm). As it requires only Python 3.7, I expect it works on Debian 10 (Buster) or later. I presume it works on other Debian-based distributions like Ubuntu.

## Features

- **Multiple configuration sources**: hosts files, command line args, and Caddyfiles
- **SIGHUP reload**: Live config reloading without stopping the service
- **Systemd integration**: Proper signal handling and transparent subprocess output
- **Automatic Caddyfile parsing**: Extracts `.local` hostnames from Caddy configurations
- **Process supervision**: Automatically restarts failed `avahi-publish` processes

## Configuration Sources

You can specify the name to IP mapping in various ways:

### 1. Caddyfile (`--caddyfile` + `--caddy-ip`)

Automatically extracts `.local` hostnames from Caddyfile address blocks:

```caddyfile
# Extracts: jupyter.box.local, files.box.local
jupyter.box.local:443 {
    reverse_proxy localhost:8888
}

files.box.local:80, files.box.local:443 {
    file_server
}
```

### 2. Hosts File (`-f/--file`)

Standard `/etc/hosts` format:
```
192.168.1.10    foo.local        bar.local
192.168.1.11    baz.local
# Comments are ignored
```

### 3. Command Line (`-a/--address`)

Direct specification of IP and hostnames:
```bash
./avahi-caddy-publish -a 192.168.1.10 foo.local -a 192.168.1.20 bar.local
```

## Caddyfile Parser

The Caddyfile parser uses regex patterns to extract mDNS hostnames:

### Regex Patterns

```python
# Matches Caddy address blocks ending with {
block_pattern = r'^([^#]*?)\s*\{\s*(?:#.*)?$'

# Matches valid .local hostnames (optionally with ports)
hostname_pattern = r'([a-zA-Z0-9][a-zA-Z0-9.-]*\.local)(?::\d+)?'
```

### Parser Features

- ✅ **Requires `{` on same line**: Only processes valid Caddy address blocks
- ✅ **Handles inline comments**: `host.local { # comment`
- ✅ **Ignores full-line comments**: `# commented.local {`
- ✅ **Comma-separated lists**: `host1.local, host2.local {`
- ✅ **Mixed with IPs**: `host.local, 192.168.1.1 {` (ignores IPs)
- ✅ **Port stripping**: `host.local:443` → `host.local`
- ✅ **Validation**: Hostnames must start with alphanumeric characters
- ✅ **Deduplication**: Same hostname on multiple ports = one mapping
- ✅ **Subdomain support**: `deep.sub.domain.local`

### What Gets Extracted

The simple regex parser matches certain lines and not others:

```caddyfile
# ✅ EXTRACTED
simple.local:443 {
multi1.local:80, multi2.local:443 {
mixed.local:443, 192.168.1.50:443 {  # Ignores IP
comment.local:443 { # Ignores comment
noport.local {
deep.sub.domain.local:443 {

# ❌ IGNORED  
example.com:443 {                    # Not .local
# commented.local:443 {              # Full-line comment
multiline.local:443                  # { not on same line
{
```

## SIGHUP Reload

To reload configuration files after making changes to your Caddyfile or hosts file, you can send a `SIGHUP` signal. Under systemd, the recommended way to do this is with `systemctl reload`:

```bash
sudo systemctl reload avahi-caddy-publish
```

Alternatively, you can use `pkill`:

```bash
pkill -HUP -f avahi-caddy-publish
```

**Reload behavior:**
- ✅ **Reloads**: hosts file (`-f`) and Caddyfile (`--caddyfile`)
- ✅ **Preserves**: command line arguments (`-a`)
- ✅ **Smart updates**: Only stops/starts processes for changed mappings
- ✅ **Atomic**: Failed reload doesn't affect running processes

## Output Examples

```
INFO: Loaded 9 address mappings (2 from file, 6 from Caddyfile, 1 from command line)
INFO: Started publishing box.local -> 192.168.1.100 (caddy) (PID: 12345)
INFO: Started publishing jupyter.box.local -> 192.168.1.100 (caddy) (PID: 12346)
Established under name 'box.local'
Established under name 'jupyter.box.local'
```

Subprocess output (from `avahi-publish`) appears transparently for journalctl visibility.

## Design Philosophy

- **Simple and robust**: Minimal dependencies, clear error handling
- **Transparent operation**: Subprocess output flows through unchanged  
- **Production ready**: Proper signal handling, process supervision
- **Caddyfile integration**: Automatically publishes services as you add them to Caddy
- **Multiple sources**: Flexible configuration for different use cases

This is intended for homelab setups where you want mDNS to automatically advertise services managed by Caddy reverse proxy.
