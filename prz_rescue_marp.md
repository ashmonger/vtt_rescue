---
marp: true
theme: gaia
size: 16:9
class:
paginate: true
footer: "OVH VTT 2023 - J. COLLIN - Rescue"
backgroundColor: #aaf
backgroundImage: url('https://marp.app/assets/hero-background.svg')

---

# Rescue

<span style="color:grey">by:</span> Jeremy Collin

---

## The Problems

* How do you install an OS on a remote server with a blank disk?
* How do you change your password on your remote server?

---

### What is <span style=color:#36f>Rescue</span> ?

![bg right height:4in](images/debian.svg)

* Live OS
* <span style=color:red>Debian</span> based
* Multitool @ OVH

---

### Philosphy

Three fundamental principles

* Compatibility
* Customisability
* Security

---

### Compatibility

* Homemade Kernel & Homemade Initramfs
* Have to work on EVERY single CPU and Motherboard:
  - about __*120*__ differents combinations (at least)
  - on about __*450000*__ baremetal servers
  - with a lifespan of __*15 years*__
* Debian, because, you know, debian

---

### Customizability

* Building packages per flavors (install, customer...)
* Forging *Cloud-Init*'s `user-data` per flavors and region (eu/us/ca)

---

### Security

* __*Chain-of-Trust*__
* Verify kernel's signature before boot (`ipxe img-verify`)
* Verify post-boot data's Signature before applying

---

### Build process

Everything is done with homemade CI/CD tool: *CDS*

* Compiling last LTS Kernel from sources
* Homemade <span style=color:red>Debian</span> bootstrap to initrd
* Recompile kernel to add initrd
* Packages building `rescue-packages`
* Forge `user-data` config files
* Setup tests for validation
* Create Pull-Request if validation is OK

---

### :rocket: CDS

#### Packages & ConfigFile build workflow

```text
┌──────────┐     ┌─────────────┐     ┌──────────────────┐     ┌───────────────┐    ┌────────────────┐
│          │     │             │     │                  │     │               │    │                │
│  Syntax  ├────►│  Extra-bin  ├────►│  MetaData-Cinit  ├────►│  Validations  ├───►│  Pull request  │
│          │     │             │     |                  │     │               │    │                │
└──────────┘     └─────────────┘     └──────────────────┘     └───────────────┘    └────────────────┘
```

---

### iPXE

```ipxe
#!ipxe
set datasource https://1674126508-public-ovh-baremetal.snap.mirrors.ovh.net/rescue/fr/rescue-install
imgtrust --permanent
imgfetch --name vmlinuz https://1674118098-public-ovh-kernel.snap.mirrors.ovh.net/rescue/rescue-kernel
imgfetch --name signature https://1674118098-public-ovh-kernel.snap.mirrors.ovh.net/rescue/rescue-kernel.sign
imgverify vmlinuz signature || goto verifyerror
kernel vmlinuz BOOTMAC=${netX/mac} rdinit=/sbin/init DATASOURCE=${datasource} METADATA=https://baremetal.eu.ovhapis.com/1.0/metadata-api
boot
:verifyerror
imgfetch https://baremetal.eu.ovhapis.com/1.0/metadata-api?mac=${mac}&ip=${ip}&error=imgverify
reboot
```

What we get in `/proc/cmdline`

```bash
vmlinuz BOOTMAC=aa:aa:aa:aa:aa:aa rdinit=/sbin/init
  DATASOURCE=https://1674126508-public-ovh-baremetal.snap.mirrors.ovh.net/rescue/fr/rescue-install
  METADATA=https://baremetal.eu.ovhapis.com/1.0/metadata-api
```

---

### <span style=color:darkorange>Cloud-Init</span>

> Cloud-init is the industry standard multi-distribution method for cross-platform cloud instance initialisation.

```yaml
#cloud-config
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQmPI7A4Lw== mine@pwet
disable_root: 0
apt:
  preserve_sources_list: true
  sources:
    public.ovh.baremetal:
      source: deb [arch=amd64] https://1674126508-public-ovh-baremetal.snap.mirrors.ovh.net/debian/ buster main
packages:
  - rescue-install
  - storage-legacy-utils
final_message: "The system is finally up, after $UPTIME seconds"
```

---
How to run <span style=color:darkorange>Cloud-Init</span> and check the file signature?

```bash
#!/bin/bash
set -euo pipefail
declare "$(grep -oE 'DATASOURCE=[^ ]*' /proc/cmdline)"
gpg --import /etc/ovhrc.d/*.pub
for file in user-data meta-data SHA256SUMS SHA256SUMS.sig
do
  curl -sSf "$DATASOURCE/$file" --output "/etc/ovhrc.d/$file"
done
cd /etc/ovhrc.d
gpg --verify SHA256SUMS.sig
sha256sum -c SHA256SUMS
cloud-init init --local
cloud-init init
cloud-init modules --mode=config
cloud-init modules --mode=final
```

---

### Releases with __*mirror-drive*__

* ZFS-based app that can create snapshot of data, and sort them by date or timestamp.
* Timestamp provided by CDS, from mirror-drive, and cannot be altered.

```bash
root@rescue-install-eu ~ # cat /etc/apt/sources.list
# This rescue has been built with the following mirror
deb https://20221215-21h42m21s-public-debian.snap.mirrors.ovh.net/debian buster main contrib non-free
deb https://20221215-21h42m21s-public-debian.snap.mirrors.ovh.net/debian buster-updates main contrib non-free
deb https://20221215-21h42m21s-public-debian.snap.mirrors.ovh.net/debian buster-proposed-updates main contrib non-free
deb https://20221215-21h42m21s-public-debian.snap.mirrors.ovh.net/debian buster-backports main contrib non-free
deb https://20221215-21h42m21s-public-debian.snap.mirrors.ovh.net/debian buster-backports-sloppy main contrib non-free
```

---

### __*Chain-of-Trust*__

* What is it?
  * Establishing that each component is properly validated
* How?
  * Kernel, Initrd : Signed, and verified with IPXE
  * Packages: Signed repositories
  * Metadata: Signed hashfile

This way each component is verified, and the system will not go further if something has been tamper with

---

### Tests

Two different workflows

- New kernel-initrd + already validated packages
- New packages + already validated kernel-initrd

Each component is tested independantly

---

### Other use cases

* on the fly hardware testing
* volatile infrastructure
* your imagination

---

## QUESTIONS?

---

### Links

- Slides: [https://github.com/ashmonger/vtt_rescue](https://github.com/ashmonger/vtt_rescue)
- CDS: [https://github.com/ovh/cds](https://github.com/ovh/cds)
- Marp: [https://github.com/marp-team/marp](https://github.com/marp-team/marp)

---

## Thank you
