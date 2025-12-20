# Monero-miner-installer

<img width="2040" height="2056" alt="image" src="https://github.com/user-attachments/assets/30c42503-7d96-4342-80bd-471e1df2a7ce" />


````markdown
# Monero Solo Mining Container on Incus (OpenTofu/Terraform)

This project provisions a **Debian 12 cloud container** on **Incus** and turns it into a **self-contained Monero full node + solo miner** using **Monero CLI (monerod)**. Everything is defined as code with **OpenTofu/Terraform**, making the setup reproducible and easy to redeploy.

## What this does

When you apply this configuration, it will:

1. **Create an Incus container** from `images:debian/12/cloud`
2. **Pin CPU cores** and **limit memory** for predictable performance and isolation
3. **Download and verify Monero CLI**:
   - Imports the official `binaryFate` signing key
   - Verifies the signed `hashes.txt`
   - Downloads the latest Monero CLI tarball (`linux64`) and checks its SHA256 hash
4. **Install `monerod`** under `/opt/monero` and symlink it to `/usr/local/bin/monerod`
5. **Install systemd services**:
   - `monerod.service` runs the Monero daemon (full node)
   - `monero-solo-mining.service` triggers solo mining via RPC using your wallet address and **4 threads**
6. **Autostart on boot**
7. Logs provisioning output to `/var/log/monero-provision.log`

## Technologies used

- **Incus**: system container manager (LXD-compatible ecosystem) used to run the Debian container.
- **OpenTofu / Terraform**: Infrastructure-as-Code to define and deploy the container and its configuration.
- **Cloud-init (NoCloud)**: used to bootstrap the container (packages, scripts, systemd units).
- **systemd**: service manager inside the container to run `monerod` and start mining automatically.
- **GPG (gnupg)**: verifies Monero release authenticity (signed hash list + trusted signer fingerprint).
- **Monero CLI**: official Monero binaries (specifically `monerod`) used for full node operation and mining.

## Prerequisites

- Incus installed and configured on the host
- OpenTofu or Terraform installed (`tofu` or `terraform`)
- Working Incus image remotes (`images:`) and access to `images:debian/12/cloud`

## Configuration variables

### `wallet_address` (required)
Monero address that will receive block rewards (solo mining).  
You must set it; the plan will fail if left at the placeholder value.

### `cpu_set`
Logical CPUs pinned to the container (example: `2,3,6,7` = two full physical cores on an i7-4770).

### `memory`
Container memory limit (example: `6GiB`).

## How to use

1. Put the `.tf` file in a directory.
2. Set your wallet address (recommended via CLI rather than editing the file):

```bash
tofu init
tofu apply -var="wallet_address=YOUR_XMR_ADDRESS_HERE"
````

(Use `terraform` instead of `tofu` if you prefer Terraform.)

## What to expect after deployment

* `monerod` starts immediately and begins syncing the blockchain.
* **Solo mining will only start once the node is synchronized**.

  * While syncing, `/start_mining` may return `BUSY` or mining will remain inactive.
  * This is expected behavior: mining effectively requires a synced node.

## Verification / health checks

### 1) Check cloud-init status

```bash
incus exec monero-miner -- cloud-init status --long
```

### 2) Check Monero daemon service

```bash
incus exec monero-miner -- sudo systemctl status monerod --no-pager -l
```

### 3) Check sync progress via RPC

```bash
incus exec monero-miner -- bash -lc \
'curl -s http://127.0.0.1:18081/get_info | egrep "\"height\"|\"target_height\"|\"synchronized\"|\"busy_syncing\""'
```

You want to eventually see:

* `"synchronized": true`

### 4) Check mining status

```bash
incus exec monero-miner -- bash -lc \
'curl -s http://127.0.0.1:18081/mining_status | egrep "\"active\"|\"threads_count\"|\"address\"|\"speed\"|\"status\""'
```
or you can directly go though the container by :
```
incus shell monero-miner
monerod status
monerod mining_status
```


Once synced and mining started successfully, you should see:

* `"active": true`
* `"threads_count": 4`
* `"address": "<your wallet address>"`
* `"speed": <non-zero>`

### 5) Provisioning logs

```bash
incus exec monero-miner -- sudo tail -n 200 /var/log/monero-provision.log
```

## Notes / operational considerations

* **Solo mining is probabilistic** (“lottery”): with low hashrate it may take a very long time to find a block.
* Blockchain sync can take significant time and disk space.
* CPU pinning helps avoid interfering with other workloads on the host.
* This setup binds the Monero RPC to `127.0.0.1` **inside the container**, reducing exposure.

## Security

* The Monero binaries are verified using:

  * a hard-checked signer fingerprint
  * a signed `hashes.txt` file
  * SHA256 hash verification of the downloaded tarball
* RPC is not exposed outside the container unless you explicitly change configuration.

If you looking to get the best anonymity possible while running a node you can follow this guide to set-up proxies : https://monero.fail/opsec

## Troubleshooting quick tips

* If mining shows inactive:

  * verify the node is fully synced (`synchronized: true`)
  * check `monero-solo-mining.service` logs:

    ```bash
    incus exec monero-miner -- sudo journalctl -u monero-solo-mining -n 200 --no-pager
    ```

* If `monerod` is not running:

  ```bash
  incus exec monero-miner -- sudo systemctl restart monerod
  incus exec monero-miner -- sudo journalctl -u monerod -n 200 --no-pager
  ```
