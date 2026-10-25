# AdGuard Home on Termux + Debian (Non-Root Android)

A step-by-step guide to running AdGuard Home on an Android phone using Termux and Debian through `proot-distro`, without rooting the phone or using Docker.

## 1. Overview

This setup runs AdGuard Home as a standalone Linux application inside Debian, with Termux providing the Android terminal environment.

**Tested configuration**

| Component | Tested setup |
|---|---|
| Android | Android 9 |
| Device architecture | ARM64 / AArch64 |
| Terminal | Termux from F-Droid |
| Linux environment | Debian 13 (Trixie) |
| AdGuard Home | v0.107.79 |
| Container technology | `proot-distro`, not Docker |
| Dashboard | `http://PHONE_IP:3000` |
| DNS listener | `PHONE_IP:5353` |
| Root access | Not required |

> **Important limitation:** This configuration runs DNS on port `5353`, not the standard DNS port `53`. It is suitable for testing and applications that support custom DNS ports. Most operating systems and home routers expect DNS on port 53, so this setup is not automatically a drop-in DNS server for your entire home network.

## 2. How it works

```text
Android phone
└── Termux
    ├── Persistent files: ~/adguard/
    └── Debian via proot-distro
        └── AdGuard Home
            ├── Dashboard: TCP 3000
            ├── DNS: UDP/TCP 5353
            ├── Configuration
            └── Filter lists and working data
```

The AdGuard Home executable is installed inside Debian. Its configuration and working data are stored in a Termux directory mounted into Debian, so they can survive a Debian environment restart or recreation, provided the Termux files are preserved.

`proot-distro` is not a full virtual machine or a real root environment. It does not grant Android root privileges or normal Linux capabilities.

## 3. Install Termux and prepare the environment

Install Termux from a trusted source, preferably [F-Droid](https://f-droid.org/packages/com.termux/).

Open Termux and update the packages:

```bash
pkg update
pkg upgrade
```

Install the required utilities:

```bash
pkg install -y proot-distro curl wget tar coreutils
```

Install Debian:

```bash
proot-distro install debian
```

Create persistent folders in Termux:

```bash
mkdir -p "$HOME/adguard/conf"
mkdir -p "$HOME/adguard/work"
mkdir -p "$HOME/adguard/logs"
```

These folders will hold the AdGuard Home configuration, working data, and any additional logs you choose to store.

## 4. Enter Debian and install prerequisites

Enter Debian from the normal Termux shell:

```bash
proot-distro login --bind "$HOME/adguard:/opt/adguard-persistent" debian
```

**Important:** Run `proot-distro login` from Termux, not from a shell already inside Debian. Running it inside a `proot` session causes an error.

Update Debian and install the required packages:

```bash
apt-get update
apt-get install -y ca-certificates apache2-utils curl dnsutils
update-ca-certificates
```

These packages provide:

- `ca-certificates`: TLS certificate trust for HTTPS connections.
- `apache2-utils`: provides `htpasswd` for creating a bcrypt password hash.
- `curl`: tests HTTPS connectivity.
- `dnsutils`: provides `dig` for DNS testing.

Check the certificate bundle:

```bash
ls -lh /etc/ssl/certs/ca-certificates.crt
```

It should exist and have a non-zero size.

## 5. Identify the phone's Wi-Fi IP address

AdGuard Home will be configured to listen on the phone's Wi-Fi IP address.

Find the phone's current IP address in Android's Wi-Fi network details. For example:

```text
192.168.2.10
```

This is an example address, not a universal value. Use the actual IP address assigned to your phone.

For this guide, commands use `192.168.2.10`. Replace it with your phone's address everywhere it appears.

If the address changes after reconnecting to Wi-Fi, update the configuration. A DHCP reservation in the router can help keep the address stable.

## 6. Download AdGuard Home

Check the official release page first:

https://github.com/AdguardTeam/AdGuardHome/releases

This guide was tested with **v0.107.79**. The following commands use that version as a reproducible baseline; check the official releases for newer stable versions before deploying elsewhere.

Inside Debian:

```bash
mkdir -p /opt/adguard
cd /opt/adguard
```

Download the Linux ARM64 archive:

```bash
curl -fL -o AdGuardHome_linux_arm64.tar.gz \
  https://github.com/AdguardTeam/AdGuardHome/releases/download/v0.107.79/AdGuardHome_linux_arm64.tar.gz
```

Verify the downloaded archive's SHA-256 against the checksum published for the exact asset on the official GitHub release page.

Calculate the local checksum:

```bash
sha256sum AdGuardHome_linux_arm64.tar.gz
```

**Do not skip verification:** compare the complete result with the official checksum for that exact release asset. Do not assume a checksum from another version or archive applies.

Extract the archive:

```bash
tar -xzf AdGuardHome_linux_arm64.tar.gz
```

Verify the executable:

```bash
/opt/adguard/AdGuardHome/AdGuardHome --version
```

Confirm that the reported version and architecture match the downloaded release and your device.

## 7. Configure the dashboard and DNS listener

On non-root Android, binding to privileged port 53 is generally unavailable to an ordinary process. Configure AdGuard Home to use port `5353` for DNS.

The configuration file will be:

```text
/opt/adguard-persistent/conf/AdGuardHome.yaml
```

If the file does not exist, launch AdGuard Home once to generate its initial configuration:

```bash
/opt/adguard/AdGuardHome/AdGuardHome \
  --config /opt/adguard-persistent/conf/AdGuardHome.yaml \
  --work-dir /opt/adguard-persistent/work \
  --no-check-update
```

If it exits with a privileged-port permission error, that is expected before changing the DNS port. If it remains running, stop it with `Ctrl+C` before editing the configuration.

Open the file:

```bash
nano /opt/adguard-persistent/conf/AdGuardHome.yaml
```

Back up the file first if it already exists:

```bash
cp -p /opt/adguard-persistent/conf/AdGuardHome.yaml \
  /opt/adguard-persistent/conf/AdGuardHome.yaml.bak
```

Update the existing `http` and `dns` sections to use these values, keeping all unrelated settings:

```yaml
http:
  address: 192.168.2.10:3000

dns:
  bind_hosts:
    - 192.168.2.10
  port: 5353
```

Replace the example IP with your phone's actual Wi-Fi IP. Do not create duplicate `http:` or `dns:` keys if those sections already exist.

Validate the configuration:

```bash
/opt/adguard/AdGuardHome/AdGuardHome \
  --check-config \
  --config /opt/adguard-persistent/conf/AdGuardHome.yaml \
  --work-dir /opt/adguard-persistent/work
```

Proceed only when the output confirms that the configuration file is valid.

## 8. Configure dashboard authentication

Do not leave the dashboard without authentication when it is accessible from your LAN.

Generate a bcrypt password hash:

```bash
htpasswd -B -C 10 -n adguardadmin
```

Enter a strong password at the prompts. The password is not displayed while typing.

The command prints a line similar to:

```text
adguardadmin:$2y$10$...
```

Use your actual complete hash; the text above is only an example.

Stop AdGuard Home before editing the configuration. Back up the YAML file if you have not already done so:

```bash
cp -p /opt/adguard-persistent/conf/AdGuardHome.yaml \
  /opt/adguard-persistent/conf/AdGuardHome.yaml.bak
```

Open the configuration:

```bash
nano /opt/adguard-persistent/conf/AdGuardHome.yaml
```

Find the existing top-level `users:` section. Replace an empty `users: []` section, or update the existing user list, so it contains your username and bcrypt hash:

```yaml
users:
  - name: "adguardadmin"
    password: "$2y$10$PASTE_YOUR_COMPLETE_BCRYPT_HASH_HERE"
```

Replace the example hash with the full hash generated on your device. Keep the hash quoted and do not create a second `users:` key.

Save the file and validate it again:

```bash
/opt/adguard/AdGuardHome/AdGuardHome \
  --check-config \
  --config /opt/adguard-persistent/conf/AdGuardHome.yaml \
  --work-dir /opt/adguard-persistent/work
```

AdGuard Home's configuration documentation describes `users` entries as usernames with bcrypt-encrypted passwords:

https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration

## 9. Start AdGuard Home

Run:

```bash
/opt/adguard/AdGuardHome/AdGuardHome \
  --config /opt/adguard-persistent/conf/AdGuardHome.yaml \
  --work-dir /opt/adguard-persistent/work \
  --no-check-update
```

Keep this terminal session open while using the service.

Look for startup messages showing the web interface and DNS listeners. Expected addresses are:

```text
http://192.168.2.10:3000
192.168.2.10:5353 (UDP)
192.168.2.10:5353 (TCP)
```

Use your actual phone IP address if it differs from the example.

## 10. Verify the dashboard

From a browser on the phone or another device connected to the same Wi-Fi, open:

```text
http://192.168.2.10:3000
```

Confirm that:

1. The AdGuard Home dashboard or login page appears.
2. Authentication is required.
3. You can sign in using the configured username and password.
4. The dashboard opens successfully.

Do not expose this HTTP dashboard directly to the public internet. For remote access, use a properly secured private network or VPN.

## 11. Fix and test HTTPS certificate trust

If the AdGuard Home logs show errors such as:

```text
x509: certificate signed by unknown authority
```

verify that Debian's CA certificates are installed and refreshed:

```bash
apt-get install -y ca-certificates
update-ca-certificates
```

Test HTTPS connectivity to Quad9:

```bash
curl -I --max-time 15 https://dns10.quad9.net/dns-query
```

An HTTP error response such as `400` can be normal for a simple HEAD request to a DNS-over-HTTPS endpoint. The important point for this test is that TLS certificate verification succeeds without an `x509` error.

If you configure Quad9 as an upstream in AdGuard Home, inspect the AdGuard Home runtime output afterward to confirm that upstream requests and filter downloads work.

## 12. Test DNS resolution

Install `dnsutils` if necessary:

```bash
apt-get install -y dnsutils
```

Query AdGuard Home directly:

```bash
dig @192.168.2.10 -p 5353 example.com
```

A successful response should contain:

```text
status: NOERROR
```

It should also show one or more returned IP addresses in the `ANSWER SECTION`, and the server line should identify your phone's IP with port `5353`.

This confirms DNS resolution through AdGuard Home. It does not, by itself, prove that every ad or tracker will be blocked.

## 13. Verify filtering

Open the dashboard and check the filtering configuration.

Confirm that a filter list has downloaded and updated successfully. The AdGuard Home logs should indicate that the filter was updated and report a rule count.

You can also inspect the query log in the dashboard after making DNS queries.

If a filter download fails, check:

- Whether Debian has a working internet connection.
- Whether CA certificates are installed.
- Whether the configured upstream DNS server is reachable.
- Whether the filter URL is accessible.
- The runtime log messages for the specific failure.

## 14. Persistent storage and backups

The Termux directory structure used in this setup is:

```text
$HOME/adguard/
├── conf/
│   ├── AdGuardHome.yaml
│   └── AdGuardHome.yaml.bak
├── work/
│   └── data/
└── logs/
```

Inside Debian, the persistent directory is mounted at:

```text
/opt/adguard-persistent
```

The configuration and working data are important. The working data may include filter lists, sessions, and other runtime state.

Back up the configuration before changing it. Keep copies of the configuration and any data you need to retain. Do not delete the working data directory merely to troubleshoot a startup problem.

The executable is located separately in Debian at:

```text
/opt/adguard/AdGuardHome/AdGuardHome
```

## 15. Common problems

### `proot-distro` reports that it was run inside a proot session

Exit Debian until you reach the normal Termux shell, then run:

```bash
proot-distro login --bind "$HOME/adguard:/opt/adguard-persistent" debian
```

### AdGuard Home refuses to bind to port 53

Use port `5353` for this non-root Android setup. Do not assume that Debian's simulated root account grants real Android root privileges.

### Dashboard is unreachable

Check that:

- AdGuard Home is running.
- The browser is using the correct phone IP and port `3000`.
- The phone and browser device are connected to the same Wi-Fi.
- The phone's Wi-Fi IP has not changed.
- The `http.address` setting uses the correct IP.

### DNS test times out

Check that the AdGuard Home process is running and that the `dns.bind_hosts` and `dns.port` settings match the address and port used by `dig`.

### TLS certificate errors appear

Install and update `ca-certificates` inside Debian, then retest HTTPS connectivity.

### AdGuard Home stops when Termux is closed

The foreground process depends on the terminal session and Android's process-management behavior. Battery optimization and background restrictions may stop Termux. This playbook does not configure an automatic startup service or guarantee continuous availability.

## 16. Limitations and security notes

- **Port 53:** Most clients expect standard DNS on port 53. A listener on `5353` cannot automatically replace a router's or operating system's ordinary DNS server.
- **No root:** `proot-distro` does not provide actual Android root access or kernel-level networking capabilities.
- **Wi-Fi IP changes:** A changed IP can make the dashboard and DNS listener unreachable until the configuration is updated.
- **Availability:** Android may suspend or terminate background processes. This is not equivalent to a dedicated always-on DNS server.
- **Dashboard security:** Use a strong password, retain authentication, and do not expose the HTTP dashboard directly to the internet.
- **Data safety:** Preserve the Termux-backed configuration and working directories, especially when upgrading or troubleshooting.

## Official references

- [AdGuard Home releases](https://github.com/AdguardTeam/AdGuardHome/releases)
- [AdGuard Home project and documentation](https://github.com/AdguardTeam/AdGuardHome)
- [AdGuard Home configuration reference](https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration)
- [Termux](https://termux.dev/)
- [proot-distro](https://github.com/termux/proot-distro)

---

**Test status:** The procedure above is based on a working installation of AdGuard Home v0.107.79 on Android 9 / ARM64 using Termux and Debian 13. Your device, Android version, network configuration, and future AdGuard Home releases may behave differently. Verify each step and check the official release documentation before deploying it on another device.
