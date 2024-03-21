# Introduction

When Fedora (N+1) is in final freeze, some packages in Fedora (N) will sort as newer than packages in Fedora (N+1). We prevent downgrades by fast-tracking any packages that would violate this "no downgrade" rule.

Example:
```
linux-firmware 20240312-1.fc39 -> 20240220-1.fc40
```

# Finding downgraded packages in `next-devel`

To find the downgraded packages in `next-devel`, which tracks Fedora (N+1), we must compare the packages to those in `testing-devel` since that stream tracks Fedora (N).

``` bash
fedora-version=40 #replace this with the current Fedora (N+1) version
podman run -i --rm registry.fedoraproject.org/fedora:${fedora-version} <<EOF
    dnf install -y fedora-repos-ostree ostree rpm-ostree
	mkdir /tmp/repo
	cd /tmp/repo
	ostree init --mode=archive --repo=.
	cat /etc/ostree/remotes.d/fedora-compose.conf >> ./config
	ostree pull --commit-metadata-only --depth=0 fedora-compose:fedora/x86_64/coreos/testing-devel
	ostree pull --commit-metadata-only --depth=0 fedora-compose:fedora/x86_64/coreos/next-devel
	rpm-ostree --repo=. db diff fedora-compose:fedora/x86_64/coreos/testing-devel fedora-compose:fedora/x86_64/coreos/next-devel | sed -ne '/Downgraded/,/Removed/{/Downgraded/N;p}'
EOF
```

```text
...
...
Downgraded:
  amd-gpu-firmware 20240312-1.fc39 -> 20240220-1.fc40
  amd-ucode-firmware 20240312-1.fc39 -> 20240220-1.fc40
  atheros-firmware 20240312-1.fc39 -> 20240220-1.fc40
  bind-libs 32:9.18.24-1.fc39 -> 32:9.18.21-4.fc40
  bind-license 32:9.18.24-1.fc39 -> 32:9.18.21-4.fc40
  bind-utils 32:9.18.24-1.fc39 -> 32:9.18.21-4.fc40
  brcmfmac-firmware 20240312-1.fc39 -> 20240220-1.fc40
  conmon 2:2.1.10-1.fc39 -> 2:2.1.8-4.fc40
  container-selinux 2:2.230.0-1.fc39 -> 2:2.229.0-2.fc40
  crun 1.14.4-1.fc39 -> 1.14.3-1.fc40
  crun-wasm 1.14.4-1.fc39 -> 1.14.3-1.fc40
  device-mapper-persistent-data 1.0.12-1.fc39 -> 1.0.11-3.fc40
  elfutils-default-yama-scope 0.191-2.fc39 -> 0.190-6.fc40
  elfutils-libelf 0.191-2.fc39 -> 0.190-6.fc40
  elfutils-libs 0.191-2.fc39 -> 0.190-6.fc40
  fwupd 1.9.15-1.fc39 -> 1.9.14-1.fc40
  intel-gpu-firmware 20240312-1.fc39 -> 20240220-1.fc40
  libnfsidmap 1:2.6.4-0.rc5.fc39 -> 1:2.6.4-0.rc4.fc40
  libusb1 1.0.27-1.fc39 -> 1.0.26-6.fc40
  linux-firmware 20240312-1.fc39 -> 20240220-1.fc40
  linux-firmware-whence 20240312-1.fc39 -> 20240220-1.fc40
  mt7xxx-firmware 20240312-1.fc39 -> 20240220-1.fc40
  nfs-utils-coreos 1:2.6.4-0.rc5.fc39 -> 1:2.6.4-0.rc4.fc40
  nvidia-gpu-firmware 20240312-1.fc39 -> 20240220-1.fc40
  realtek-firmware 20240312-1.fc39 -> 20240220-1.fc40
  rpm-ostree 2024.3-3.fc39 -> 2024.3-2.fc40
  rpm-ostree-libs 2024.3-3.fc39 -> 2024.3-2.fc40
  socat 1.8.0.0-2.fc39 -> 1.7.4.4-5.fc40
  vim-data 2:9.1.181-1.fc39 -> 2:9.1.113-1.fc40
  vim-minimal 2:9.1.181-1.fc39 -> 2:9.1.113-1.fc40

```


# Determine Bodhi Update For Downgraded Packages

### Listing Package Builds
For each package, we must find the latest bodhi update. Use the following command to find the list of builds for the package, then later we'll use the output to find the bodhi update.
``` bash
$ bodhi updates query --releases f40 --packages ${package}
```

### Finding The Source RPM For A Sub Package
If this command returns "0 updates found", then the package is likely a sub package. We need to find the bodhi update for the base package. We can use `dnf info $package` to find the source rpm for a given package.

``` bash
$ dnf info realtek-firmware

Fedora 40 - x86_64                               16 MB/s |  33 MB     00:02    
Fedora 40 openh264 (From Cisco) - x86_64        2.0 kB/s | 1.8 kB     00:00    
Fedora 40 - x86_64 - Updates                    261  B/s | 134  B     00:00    
Fedora 40 - x86_64 - Test Updates               2.4 MB/s | 3.7 MB     00:01    
Available Packages
Name         : realtek-firmware
Version      : 20240312
Release      : 1.fc40
Architecture : noarch
Size         : 2.4 M
Source       : linux-firmware-20240312-1.fc40.src.rpm
Repository   : updates-testing
Summary      : Firmware for Realtek WiFi/Bluetooth adapters
URL          : http://www.kernel.org/
License      : Redistributable, no modification permitted
Description  : Firmware for Realtek WiFi/Bluetooth adapters
```

The source rpm is listed in the `Source` field of the output. Use the name of the source package to find the list of bodhi updates.

```bash
$ bodhi updates query --releases f40 --package linux-firmware

 linux-firmware-20240312-1.fc40           rpm        testing   2024-03-13 (8)
 linux-firmware-20240220-1.fc40           rpm        stable    2024-02-21 (29)
 linux-firmware-20240115-2.fc40           rpm        stable    2024-01-18 (62)
 linux-firmware-20240115-1.fc40           rpm        stable    2024-01-16 (65)
 linux-firmware-20231211-1.fc40           rpm        stable    2023-12-13 (98)
 linux-firmware-20231111-1.fc40           rpm        stable    2023-11-15 (126)
 linux-firmware-20231030-1.fc40           rpm        stable    2023-10-31 (141)
 linux-firmware-20230919-1.fc40           rpm        stable    2023-09-20 (183)
 linux-firmware-20230804-153.fc40         rpm        stable    2023-08-10 (223)
 linux-firmware-20230804-152.fc40         rpm        stable    2023-08-09 (224)
10 updates found (10 shown)
```

Look for the build that matches the package in Fedora (N), in this case it's `linux-firmware-20240312-1.fc40`.
Note that the latest build is in the testing state because final-freeze doesnt allow a package to enter stable state yet.

# Use The Latest Build To Find The Bodhi Update

To find the bodhi update for the latest build, use the following command.

```bash
bodhi updates query --releases f40 --builds ${latest_update}
```

The `Update ID` field is what we are looking for.

```bash
$ bodhi updates query --releases f40 --builds linux-firmware-20240312-1.fc40 | grep "Update ID"

Update ID: FEDORA-2024-e62bcaf172
```

Then, you can view the bodhi update in a browser:

https://bodhi.fedoraproject.org/updates/FEDORA-2024-e62bcaf172

### Inspecting The Bodhi Update

It's a good proctice to ensure that the latest update doesn't have negative karma. Updates that have negative karma could intoruduce bugs or regressions into the build, so it might be best ot wait until the next update for the maintainer to resolve any issues.

# Fast Track the Packages and Open a PR against f-c-c

Fast track the latest bodhi update for each downgraded package in Fedora (N+1) using the `overrides.py` script in the fedora-coreos-config repository. However, the script cannot handle arch-specific updates, so manual fast-tracking may be required.

Once all packages have been fast-tracked, open a PR against the next-devel branch of fedora-coreos-config. See this example PR: https://github.com/coreos/fedora-coreos-config/pull/2696
