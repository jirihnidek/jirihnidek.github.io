---
title: Virtualization and libVirt
layout: wiki
tags: linux, virtualization, libvirt
language: Czech
---

Když chceme začít virtualizovat, tak je nejprve nutné si nějaký virtualizovaný OS nainstalovat. Například můžeme použít 64 bitový CentOS Linux virtualizovaný pomocí technologie XEN.

## XEN ##

XEN použijeme tehdy, pokud chceme použít paravirtulizované prostředí:

```bash
virt-install --paravirt \
     --name pokus \
     --ram 512 \
     --file=/home/xen/pokus.img \
     --file-size 20 \
     --nonsparse \
     --nographics \
     --os-type=linux --os-variant=rhel5.4 \
     --location  http://ftp.cvut.cz/centos/5.5/os/x86_64/ \
     --mac 00:16:3e:00:11:22 --bridge xenbr0
```

Význam jednotlivých voleb a parametrů je následující:

* _--paravirt_ ... paravirtualizace pomocí Xen je naše volba
* _--name pokus_ ... název virtuálního stroje
* _--ram 512_ ... virtuálnímu stroji vyhradíme 512MB RAM
* _--file=/home/xen/pokus.img_ ... disk virtuální stroj uložíme do souboru 
* _--file-size 20_ ... kterýžto bude mít 20GB
* _--nonsparse_ ... a nebude obsahovat díry, takže IO operace budou rychlé jako z praku
* _--nographics_ ... nepotřebujeme žádnou grafiku
* _--os-type=linux --os-variant=rhel5_ ... prý nepatrně zlepšuje výkon
* _--location_  http://ftp.cvut.cz/centos/5.3/os/x86_64/ ... umístění instalačního média
* _--mac 00:16:3e:00:11:22_ ... MAC adresa virtuální síťové karty (Xen má rozsah: **00:16:3e:**..:..:..)
* _--bridge xenbr0_ ... a jelikož chceme, aby byl virtuální stroj viditelný na vnější síti, tak budeme bridgovat

### Windows XP ###

Pokud chcete použít Xen k virtualizaci Windows XP, tak je potřeba instalovat Windows pomocí obrazu instalačního CD Windows XP SP2 (nikoliv SP3 ... s ním instalace selže). Vlastní příkaz potom může vypadat následovně:

```bash
virt-install --hvm \
     --name windows_xp \
     --ram 512 \
     --file=/home/xen/windows_xp.img \
     --file-size 20 \
     --nonsparse \
     --os-type=windows \
     --os-variant=winxp \
     --cdrom /root/iso/windows_xp_pro_sp2.iso \
     --mac 00:16:3e:12:34:56 \
     --bridge xenbr0
```

## KVM ##

Pokud chceme použít plnou virtualizaci, tak můžeme použít KVM. Před použitím KVM je potřeba ověřit, že procesor má podporu pro virtualizaci. Když vypíšeme obsah souboru _/proc/cpuinfo_, tak procesor(y) by měli mít příznak _vmx_ (Intel) nebo _vms_ (AMD). Potom je také většinou potřeba podporu pro virtualizaci zapnout v BIOSu. Bývá většinou vypnutá. To, jestli je zapnutá nebo ne ne, lze poznat pomocí příkazu:

```bash
dmesg | grep kvm
```

V případě, že je v podpora pro virtualizaci v BIOSu vypnutá, tak to vrátí

    kvm: disabled by bios

### Síťový most ###

Než nainstalujeme KVM, tak je nejprve potřeba nastavit síťový most, protože KVM to za nás na rozdíl od Xenu neudělá. Je potřeba vytvořit soubor:

    /etc/sysconfig/network-scripts/ifcfg-br0

který bude obsahovat následující konfiguraci:

    DEVICE="br0"
    TYPE="Bridge"
    BOOTPROTO="dhcp"
    ONBOOT="yes"
    DELAY="0"
    NM_CONTROLLED="no"

Následně je potřeba upravit konfigurační soubor odpovídající síťovému zařízení, které chceme přemostit. V našem případě to bylo zařízeni _em1_:

    /etc/sysconfig/network-scripts/ifcfg-em1

kde bylo potřeba upravit/přidat tyto řádky:

    NM_CONTROLLED="no" 
    BRIDGE="br0"

Nakonec je potřeba restartovat nastavení sítě pomocí

```bash
service network restart
```

### Vlastní instalace ###

Vlastní instalaci lze provést několika způsoby. My si ukážeme instalaci pomocí příkazu _virt-install_:

```bash
virt-install \
     --accelerate --hvm \
     --connect qemu:///system \
     --name pokus \
     --ram 512 \
     --file=/home/kvm/pokus.img \
     --file-size 20 \
     --nonsparse \
     --vnc --noautoconsole \
     --os-type=linux --os-variant=rhel6.3 \
     --location  http://ftp.cvut.cz/centos/6.3/os/x86_64/ \
     --mac 52:54:00**:00:11:22 --network=bridge:br0
```

Tento příkaz pouze spustí instalaci. Vlastní průběh instalace je nutné sledovat a provádět například přes virt-manager, který se připojí k hypervizorovi (lze i vzdáleně pomocí ssh).

**Poznámky:**
* Není možné zároveň používat paravirtualizační řešení postavené na XENu a plnou virtualizaci postavenou na KVM.
* Význam jednotlivých voleb a parametrů je stejný jako u prvního případu. Jsou zde pouze zvýrazněny rozdíly proti XENu.
* Výchozím hypervizorem je na RedHatu 5 XEN a pokud nechcete při každém použití příkazu _virsh_ nastavovat jako hypervizor KVM, tak je vhodné nastavit si systémovou proměnou:

```bash
export VIRSH_DEFAULT_CONNECT_URI="qemu:///system"
```

## Rozsahy MAC adres ##

Každý výrobce síťových karet má přidělený nějaký rozsah nebo rozsahy MAC adres. Podobně je to i s tvůrci virtualizačních řešení.

* VMWare/ESXi: 00:50:56:00:00:00 / 00:50:56:3E:FF:FF 
* XEN: 00:16:3E:00:00:00 / 00:16:3E:FF:FF:FF 
* VirtualBox: 08:00:27:00:00:00 / 08:00:27:FF:FF:FF
* KVM: 52:54:00:00:00:00 / 52:54:00:FF:FF:FF

Bohužel zde už vůbec neexistují mechanismy, které by zamezili tomu, aby na světě nebyli dva virtuální stroje se stejnou MAC adresou. V každém případě je na vás, abyste zajistili unikátnost všech virtualizovaných strojů ve vaší síti.

### Generování náhodných MAC adres ###

Pro KVM lze náhodnou MAC adresu vygenerovat pomocí:

```bash
printf '52:54:00:%02X:%02X:%02X\n' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256))
```
