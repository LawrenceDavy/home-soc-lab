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

*This log will continue as I work through the remaining milestones and document any additional issues I encounter.*
