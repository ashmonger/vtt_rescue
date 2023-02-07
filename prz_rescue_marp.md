---
marp: true
theme: uncover
class: invert
//paginate: true
//backgroundColor: #fff
//backgroundImage: 'background/hero-background.svg'
---
# Rescue

by: Jeremy Collin

---
## The Problems

* How do you install an OS on a remote server with a blank disk?
* How do you change your password on your remote server?

---
## What is *Rescue* ?

 * Live OS
 * Debian based
 * Multitool @ OVH

---
## Philosphy

Three fundamental principles
* Compatibility
* Customisability
* Seciurity

---
### Compatibility
 * Homemade Kernel & Homemade Initramfs
 * Kernel working on EVERY single CPU and Motherboard
    * about __*120*__ differents combinations (at least)
    * on about __*450000*__ baremetal servers
    * with a lifespan of __*15 years*__
 * Debian, because, you know, debian

---
### Customizability
 * Building packages per flavors (install, customer...)
 * Forging *Cloud-Init*'s `user-data` per flavors and region (eu/us/ca)

---
### Security
 * Chain-of-Trust
 * Verify kernel's signature before boot (`ipxe img-verify`)
 * Verify post-boot data's Signature before applying

---
## Build process

* Compiling last LTS Kernel from sources
* Homemade debian bootstrap to initrd
* Recompile kernel to add initrd
* Packages building `rescue-packages`
* Forge `user-data` config files
* Setup tests for validation
* Create Pull-Request if validation is OK

Everything is done with homemade CI/CD tool: *CDS*

---
##3 Packages & Config file build workflow 

```
┌──────────┐     ┌─────────────┐     ┌──────────────────┐     ┌───────────────┐    ┌────────────────┐
│          │     │             │     │                  │     │               │    │                │
│  Syntax  ├────►│  Extra-bin  ├────►│  MetaData-Cinit  ├────►│  Validations  ├───►│  Pull request  │
│          │     │             │     |                  │     │               │    │                │
└──────────┘     └─────────────┘     └──────────────────┘     └───────────────┘    └────────────────┘
```

---
### iPXE

```
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

```
vmlinuz BOOTMAC=aa:aa:aa:aa:aa:aa rdinit=/sbin/init
  DATASOURCE=https://1674126508-public-ovh-baremetal.snap.mirrors.ovh.net/rescue/fr/rescue-install
  METADATA=https://baremetal.eu.ovhapis.com/1.0/metadata-api
```

---
### Cloud-Init

>Cloud-init is the industry standard multi-distribution method for cross-platform cloud instance initialisation.
It is supported across all major public cloud providers, [...], and bare-metal installations.
(R) Canonical

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