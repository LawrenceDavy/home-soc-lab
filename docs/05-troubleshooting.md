# Troubleshooting Log

> These are issues I ran into while building this lab on 2 August 2026.
> Everything here happened during the build and was documented as I fixed it.
> Nothing has been recreated or made up afterwards.

---

## 1. `bridge-utils` — target not found

**When:** Step 1, initial package installation.

**Symptom:**
```text
error: target not found: bridge-utils
```

Pacman stopped the entire installation, so none of the other packages were installed even though they were available.

**Cause:** `bridge-utils` is no longer included in the Arch Linux repositories. The old `brctl` functionality has been replaced by tools included with `iproute2`, which is already installed as part of the base system. Since one package in the command no longer existed, Pacman refused to continue.

**Fix:** I removed `bridge-utils` from the install command and ran it again.

```bash
sudo pacman -S --needed qemu-full libvirt virt-manager dnsmasq iptables-nft edk2-ovmf swtpm
```

Everything installed successfully after that. No functionality was lost because libvirt already uses the networking tools provided by `iproute2`.

---

## 2. Multiple 404 errors when installing QEMU packages

**When:** Immediately after fixing issue #1 and retrying the installation.

**Symptom:** I received a large number of `404` download errors from several different mirrors. The errors affected packages such as `qemu-system-x86`, `qemu-user`, `qemu-system-arm`, and other QEMU components, causing Pacman to fail with a transaction error.

**Cause:** My local package database was out of date. Pacman was trying to download package versions that had already been replaced on the mirrors, so every mirror correctly returned a 404 because those files no longer existed.

**Fix:** I performed a full package database refresh and system upgrade.

```bash
sudo pacman -Syyu
```

The `-yy` option forces Pacman to refresh the package databases even if it believes they're already current. After the update completed, I ran the command again and got the expected `there is nothing to do` message, confirming everything was synced correctly.

Once that finished, I reran the original installation command and all packages installed successfully.

**Note:** During the full system upgrade I hit the same issue again with a few CachyOS mirrors while downloading core packages such as `glibc`, `gcc`, and `binutils`. Running `sudo pacman -Syyu` again resolved it and allowed the update to finish cleanly.

---

## 3. VM name defaulted to `win11` instead of `win11-victim`

**When:** Step 5, just before creating the virtual machine.

**Symptom:** While checking the final summary screen, I noticed the VM name had been set to `win11` instead of `win11-victim`.

**Cause:** virt-manager automatically generated the VM name from the Windows ISO. It's easy to miss because the installer fills it in automatically.

**Fix:** Before clicking **Finish**, I changed the VM name to `win11-victim`. At the same time, I also enabled **Customise configuration before install**, which is disabled by default.

**Why this matters:** Later `virsh` commands reference the VM by name. If I'd left the default name in place, every command using `win11-victim` would have failed until I realised the mismatch.

---

## 4. Network adapter defaulted to `e1000e`

**When:** VM customisation screen before starting the installation.

**Symptom:** The virtual network adapter was configured to use the `e1000e` device model by default, even though the virtual disk had already been configured to use VirtIO.

**Cause:** virt-manager uses an emulated Intel network adapter by default because it offers good compatibility. While it works perfectly well, it isn't as efficient as the VirtIO driver and doesn't match the rest of the VM configuration.

**Fix:** I changed the NIC device model from `e1000e` to `virtio` before starting the installation. The network itself was already connected to the correct default NAT network, so no other changes were needed.

**Why this matters:** The VM would still have worked using `e1000e`, but using VirtIO for both storage and networking gives better performance and keeps the configuration consistent throughout the build.

---

## 5. UFW blocking libvirt DHCP and NAT

**When:** During the Windows 11 setup. I got to the "Let's connect you to a network" screen, but the VM couldn't get an internet connection even though the VirtIO network driver was installed.

**Symptom:** The network adapter was detected correctly, but Windows only got a self-assigned `169.254.x.x` IP address instead of one from DHCP. Running `virsh net-dhcp-leases default` showed no DHCP lease, but `tcpdump` on the host proved the DHCP requests were reaching the `virbr0` bridge.

**Cause:** After checking everything step by step, I found that UFW was blocking libvirt's virtual network in two different ways.

- UFW was blocking inbound UDP traffic on port 67, which is needed for DHCP. Since `dnsmasq` acts as the DHCP server for the virtual network, Windows could never get an IP address.
- UFW was also set to `Default: deny (routed)`, which blocked forwarded traffic. This meant that even after fixing DHCP, the VM still wouldn't be able to access the internet through NAT.

**Fix:** I only allowed the required traffic on the `virbr0` interface instead of opening it on every network interface.

```bash
sudo ufw allow in on virbr0 to any port 67 proto udp
sudo ufw allow in on virbr0 to any port 53 proto udp
sudo ufw allow in on virbr0 to any port 53 proto tcp
sudo ufw route allow in on virbr0
sudo ufw reload
```

**What I learned:** This took much longer than expected because every layer looked like it was working. I checked the bridge, confirmed `dnsmasq` was running, captured packets with `tcpdump`, and even ran `dnsmasq` in the foreground to watch what it was doing. Only after checking the raw `nftables` rules did I find that UFW was silently dropping the traffic. It was a good reminder that just because one part of the network is working doesn't mean another firewall rule isn't causing the real problem.

---

## 6. MSI Error 1719 / 0x80070534 during VirtIO Guest Tools install

**When:** The first time I tried to install `virtio-win-guest-tools.exe` after finishing the Windows setup and logging into the desktop.

**Symptom:** The installer failed almost immediately with the following errors:

```
Setup Failed... 0x80070643 - Fatal error during installation.

Error 1719. The Windows Installer Service could not be accessed.
Could not set proxy blanket on CA proxy interface.
Error: 0x80070534
```

**Cause:** The Windows account had only just been created during setup. Sometimes Windows hasn't fully finished creating and registering the new account's security information, so the Windows Installer service can't correctly apply permissions. This is a known timing issue rather than a problem with the VirtIO drivers themselves.

**Fix:** I restarted the virtual machine and ran the installer again. The installation completed successfully without making any other changes, confirming it was only a temporary Windows issue.

**Extra issue found:** While checking everything afterwards, I noticed I had accidentally named the account **ladadmin** instead of **labadmin** during Windows setup. I renamed it using:

```cmd
wmic useraccount where name='ladadmin' rename labadmin
```

Because my original clean snapshot had been taken before I noticed the typo, I deleted it and created a new baseline snapshot after the account was renamed so the lab documentation matched the actual environment.

**What I learned:** Not every installer error means something is broken. Sometimes Windows simply needs a restart to finish setting up a new user account. It also reminded me to double-check usernames before taking snapshots, as it's much easier to fix mistakes before creating restore points.


## 7. Microsoft Defender blocking the `certutil` attack simulation

**When:** While testing one of the attack simulations that used `certutil` to download a file.

**Symptom:** Running the command returned **"Access is denied"**, making it look like `certutil` itself wasn't working.

**Cause:** After testing further, I confirmed that `certutil` wasn't actually broken because commands like `certutil -?` worked normally. The real problem was Microsoft Defender. Defender has a built-in detection for the common attacker technique of using `certutil` as a downloader (a Living-off-the-Land Binary, or LOLBin). Checking Defender with `Get-MpThreatDetection` showed that it had detected the exact command I was trying to run (ThreatID **2147726914**) and blocked it before the process could even start.

**Current status:** I decided not to work around this straight away. Instead, I parked it for **Milestone 5**, where the focus is on attack simulations and detection testing. At that point I'll either create a temporary Defender exclusion for the testing window or use a different technique that produces the same type of detection without Defender stopping it immediately.

**What I learned:** At first it looked like a problem with `certutil`, but it turned out to be Defender doing exactly what it's supposed to do. This was a good reminder not to assume the first error message tells the whole story, and to check security logs before spending time troubleshooting the wrong thing.
*This log will continue as I work through the remaining milestones and document any additional issues I encounter.*
