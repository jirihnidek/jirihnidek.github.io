---
title: Resize Dell hardware RAID
layout: wiki
tags: linux, network, raid
language: Czech
---

RAID normálně nejde zvětšit, ale u Dellů šli do sebe a jejich harwarové řadiče to umožňují, takže když do RAIDU přidáte nový disk, tak je možné nechat celý RAID přeskládát.

Začneme tím, že si vytvoříme tzv. virtuální disk (nějaká Dellovská hantýrka nebo co). Zatím použijeme jenom tři fyzické disky (pdisk=0:0:8,0:0:9,0:0:10) a vytvoříme z nich RAID5. Volby stripesize, readpolicy, writepolicy a name jsou nepovinné.

    $ omconfig storage controller action=createvdisk controller=0 raid=r5 size=max pdisk=0:0:8,0:0:9,0:0:10 stripesize=64kb readpolicy=ara writepolicy=wb name="Virtual Disk 2"
    Command successful!

Jelikož už v serveru dva virtuální disky jsou, tak si vypíši obsah posledního (jsou číslovány od 0). Vytvoření RAIDU bude chvíli trvat (v tomto případě asi 5 hodin), ale mezitím se s virtuálním diskem dá normálně pracovat (formátovat, zapisovat, apod.).

    $ omreport storage vdisk controller=0 vdisk=2

    Virtual Disk 2 on Controller PERC H700 Integrated (Slot 4)
    
    Controller PERC H700 Integrated (Slot 4)
    ID                            : 2
    Status                        : Ok
    Name                          : Virtual Disk 2
    State                         : Background Initialization
    Hot Spare Policy violated     : Not Assigned
    Encrypted                     : No
    Progress                      : 8% complete
    Layout                        : RAID-5
    Size                          : 3,725.00 GB (3999688294400 bytes)
    Associated Fluid Cache State  : Not Applicable
    Device Name                   : /dev/sdc
    Bus Protocol                  : SAS
    Media                         : HDD
    Read Policy                   : Adaptive Read Ahead
    Write Policy                  : Write Back
    Cache Policy                  : Not Applicable
    Stripe Element Size           : 64 KB
    Disk Cache Policy             : Disabled

Na disku bude potřeba vytvořit nějaké ty diskové oddíly. Virtuální disk má více jak 2TB, takže je nutné sáhnout k nástroji `gparted` (`fdisk` by to nedal).

    $ parted /dev/sdc

    GNU Parted 1.8.1
    Používám /dev/sdc
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) print                                                           
    Chyba: Nemohu otevřít /dev/sdc - neznámá jmenovka disku.                 
    (parted) mklabel gpt
    (parted) mkpart xfs 0% 100%                                              
    (parted) print
    
    Model: DELL PERC H700 (scsi)
    Disk /dev/sdc: 4000GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    
    Number  Start   End     Size    File system  Name  Přepínače
     1      17,4kB  4000GB  4000GB  xfs          
    
    (parted) quit
    Informace: Nezapomeňte aktualizovat /etc/fstab, je-li to potřeba.

Vytvořený diskový oddíl zformátuje pomocí oblíbeného souborového systému

    $ mkfs.xfs -L experiment /dev/sdc1

    meta-data=/dev/sdc1      isize=256   agcount=32, agsize=30515199 blks
             =               sectsz=512  attr=0
    data     =               bsize=4096  blocks=976486368, imaxpct=25
             =               sunit=0     swidth=0 blks, unwritten=1
    naming   =version 2      bsize=4096
    log      =internal log   bsize=4096   blocks=32768, version=1
             =               sectsz=512   sunit=0 blks, lazy-count=0
    realtime =none           extsz=4096   blocks=0, rtextents=0

Vytvoříme adresář k připojení, nakonfigurujeme připojení a daný disk do něho připojíme.

    $ mkdir /experiment

    $ vim /etc/fstab

    $ cat /etc/fstab

    /dev/VolGroup00/LogVol00 /           ext3   defaults       1 1
    LABEL=/boot              /boot       ext3   defaults       1 2
    tmpfs                    /dev/shm    tmpfs  defaults       0 0
    devpts                   /dev/pts    devpts gid=5,mode=620 0 0
    sysfs                    /sys        sysfs  defaults       0 0
    proc                     /proc       proc   defaults       0 0
    /dev/VolGroup00/LogVol01 swap        swap   defaults       0 0
    /dev/sdb1                /data       xfs    defaults       0 0
    /dev/sdc1                /experiment xfs    defaults       0 0

    $ mount /experiment

Když je RAID kompletní ...

    $ omreport storage vdisk controller=0 vdisk=2

    Virtual Disk 2 on Controller PERC H700 Integrated (Slot 4)
    
    Controller PERC H700 Integrated (Slot 4)
    ID                            : 2
    Status                        : Ok
    Name                          : Virtual Disk 2
    State                         : Ready
    Hot Spare Policy violated     : Not Assigned
    Encrypted                     : No
    Layout                        : RAID-5
    Size                          : 3,725.00 GB (3999688294400 bytes)
    Associated Fluid Cache State  : Not Applicable
    Device Name                   : /dev/sdc
    Bus Protocol                  : SAS
    Media                         : HDD
    Read Policy                   : Adaptive Read Ahead
    Write Policy                  : Write Back
    Cache Policy                  : Not Applicable
    Stripe Element Size           : 64 KB
    Disk Cache Policy             : Disabled

... tak k němu můžeme přidat nepoužívaný disk:

    $ omreport storage pdisk controller=0 pdisk=0:0:11

    ID                              : 0:0:11
    Status                          : Ok
    Name                            : Physical Disk 0:0:11
    State                           : Ready
    Power Status                    : Spun Up
    Device Name                     : Not Available
    Bus Protocol                    : SAS
    Media                           : HDD
    Part of Cache Pool              : Not Applicable
    Device Life Remaining           : Not Applicable
    Remaining Rated Write Endurance : Not Applicable
    Failure Predicted               : No
    Revision                        : M440
    Driver Version                  : Not Applicable
    Model Number                    : Not Applicable
    Certified                       : Yes
    Encryption Capable              : No
    Encrypted                       : Not Applicable
    Progress                        : Not Applicable
    Mirror Set ID                   : Not Applicable
    Capacity                        : 1,862.50 GB (1999844147200 bytes)
    Used RAID Disk Space            : 0.00 GB (0 bytes)
    Available RAID Disk Space       : 1,862.50 GB (1999844147200 bytes)
    Hot Spare                       : No
    Vendor ID                       : HITACHI
    Product ID                      : HUS723020ALS640
    Serial No.                      : YGKNUBJK
    Part Number                     : TH0VYRKH1256739S1YFXA01
    Negotiated Speed                : 6.00 Gbps
    Capable Speed                   : 6.00 Gbps
    Device Write Cache              : Not Applicable
    Manufacture Day                 : 07
    Manufacture Week                : 39
    Manufacture Year                : 2013
    SAS Address                     : 5000CCA01CCEF4B5

Před samotnou rekonfigurací diskového pole se doporučuje data z disku zazálohovat na jiný disk nebo server. Pokud data zálohujete na jiný server patrně bude chtít zachovat přístupová práva a další metainformace uložené v i-uzlech. Z tohoto důvodu se doporučuje pro vlastní kopírování zaměstnat i tar. Pokud chcete kopírovat na jiný počítač bezpečně, tak je možné použít ssh. Je možné pro tyto účely použít i třeba netcat, ale následující řešení mi přišlo nejvíce robustní. Nejprve si na počítači, kde budeme zálohovat vytvoříme pojmenovanou rouru:

    $ mkdir /bacula_backup # zaloznik
    $ cd /bacula_backup # zaloznik
    $ mkfifo fifo # zaloznik

Následně můžeme začít do roury zapisovat:

    $ tar -cf - * | ssh zaloznik.nti.tul.cz "cat > /bacula_backup/fifo"

Na opačné straně se doporučuje z roury číst :-) a soubory z tar archívu rozbalovat. Tím si ušetříme čas v případě, že bude potřeba obnovit pouze některá data:

    $ cat fifo | tar x # zaloznik

Když je zálohování dokončené, tak můžeme přistoupit k vlastnímu zvětšení diskové pole. Pomocí následujícího příkazu existující pole rozšíříme (rekonfigurujeme) o jeden fyzický disk (pdisk=...:0:0:11). Během rekonfigurace se nestane nic s diskovým oddílem ani souborovým systémem. Virtuální disk lze normálně používat. Pouze IO operace jsou lehce pomalejší.

    $ omconfig storage vdisk action=reconfigure controller=0 vdisk=2 raid=r5 pdisk=0:0:8,0:0:9,0:0:10,0:0:11
    Command successful!

Rekonfigurace je krapítek časově náročnější a v tomto případě trvala asi dva dny. IO operace jsou asi o řád pomalejší. Rychlost rekonfigurace lze sice omezit, ale potom budete na výsledek čekat roky. :-)

    $ omreport storage vdisk controller=0 vdisk=2

    Virtual Disk 2 on Controller PERC H700 Integrated (Slot 4)
    
    Controller PERC H700 Integrated (Slot 4)
    ID                            : 2
    Status                        : Ok
    Name                          : Virtual Disk 2
    State                         : Reconstructing
    Hot Spare Policy violated     : Not Assigned
    Encrypted                     : No
    Progress                      : 9% complete
    Layout                        : RAID-5
    Size                          : 3,725.00 GB (3999688294400 bytes)
    Associated Fluid Cache State  : Not Applicable
    Device Name                   : /dev/sdc
    Bus Protocol                  : SAS
    Media                         : HDD
    Read Policy                   : No Read Ahead
    Write Policy                  : Write Through
    Cache Policy                  : Not Applicable
    Stripe Element Size           : 64 KB
    Disk Cache Policy             : Disabled

Po “rekonstrukci” se ještě provede nějaká inicializace. Ta trvá přibližně stejně dlouho jako rekonstrukce.

    $ omreport storage vdisk controller=0 vdisk=2

    Virtual Disk 1 on Controller PERC H700 Integrated (Slot 4)
    
    Controller PERC H700 Integrated (Slot 4)
    ID                            : 2
    Status                        : Ok
    Name                          : Virtual Disk 2
    State                         : Background Initialization
    Hot Spare Policy violated     : Not Assigned
    Encrypted                     : No
    Progress                      : 54% complete
    Layout                        : RAID-5
    Size                          : 5,587.50 GB (5999532441600 bytes)
    Associated Fluid Cache State  : Not Applicable
    Device Name                   : /dev/sdc
    Bus Protocol                  : SAS
    Media                         : HDD
    Read Policy                   : Adaptive Read Ahead
    Write Policy                  : Write Back
    Cache Policy                  : Not Applicable
    Stripe Element Size           : 64 KB
    Disk Cache Policy             : Disabled

Můžeme se přesvědčit, že přidaný disk už není `Ready`, ale už se přepnul do stavu `Online`. To znamená, že už je používaný posledním virtuálním diskem (diskovým polem).

    $ omreport storage pdisk controller=0 pdisk=0:0:11

    ID                              : 0:0:11
    Status                          : Ok
    Name                            : Physical Disk 0:0:11
    State                           : Online
    Power Status                    : Spun Up
    Device Name                     : Not Available
    Bus Protocol                    : SAS
    Media                           : HDD
    Part of Cache Pool              : Not Applicable
    Device Life Remaining           : Not Applicable
    Remaining Rated Write Endurance : Not Applicable
    Failure Predicted               : No
    Revision                        : M440
    Driver Version                  : Not Applicable
    Model Number                    : Not Applicable
    Certified                       : Yes
    Encryption Capable              : No
    Encrypted                       : Not Applicable
    Progress                        : Not Applicable
    Mirror Set ID                   : Not Applicable
    Capacity                        : 1,862.50 GB (1999844147200 bytes)
    Used RAID Disk Space            : 0.00 GB (0 bytes)
    Available RAID Disk Space       : 1,862.50 GB (1999844147200 bytes)
    Hot Spare                       : No
    Vendor ID                       : HITACHI
    Product ID                      : HUS723020ALS640
    Serial No.                      : YGKNUBJK
    Part Number                     : TH0VYRKH1256739S1YFXA01
    Negotiated Speed                : 6.00 Gbps
    Capable Speed                   : 6.00 Gbps
    Device Write Cache              : Not Applicable
    Manufacture Day                 : 07
    Manufacture Week                : 39
    Manufacture Year                : 2013
    SAS Address                     : 5000CCA01CCEF4B5

Po rekonfiguraci RAIDu můžeme vidět, že se velikost virtuálního disku zvětšila z 3.7GB na 5.6GB.

    $ omreport storage vdisk controller=0 vdisk=2

    Virtual Disk 2 on Controller PERC H700 Integrated (Slot 4)
    
    Controller PERC H700 Integrated (Slot 4)
    ID                          : 2
    Status                      : Ok
    Name                        : Virtual Disk 2
    State                       : Ready
    Hot Spare Policy violated   : Not Assigned
    Encrypted                   : No
    Layout                      : RAID-5
    Size                        : 5,587.50 GB (5999532441600 bytes)
    Associated Fluid Cache State  : Not Applicable
    Device Name                 : /dev/sdc
    Bus Protocol                : SAS
    Media                       : HDD
    Read Policy                 : Adaptive Read Ahead
    Write Policy                : Write Back
    Cache Policy                : Not Applicable
    Stripe Element Size         : 64 KB
    Disk Cache Policy           : Disabled

Samotný diskový oddíl a souborový systém má ale stále původní velikost:

    $ df -h

    Filesystem          Size  Used Avail Use% Mounted on
    ...
    /dev/sdc1           3,7T   57G  3,6T   2% /experiment

Pro následující operace je potřeba souborový systém nejprve odpojit:

    $ umount /experiment/

Kdyby se odpojení nedařilo, tak je možné získat pachatele pomocí příkazů:

    $ fuser -m /dev/sdc1

    /dev/sdc1: 579

Takže je nejprve nutné změnit velikost diskového oddílu a následně souborového systému. Před tím, než můžeme začít něco zvětšovat, tak musíme Linuxu říci, že nějaký `omconfig` disk zvětšil (nedělá to automaticky). To můžeme udělat dvěma způsoby. Buď restartujeme celý systém, nebo elegantněji spuštěním příkazu:

    $ blockdev --rereadpt /dev/sdc

Velikost diskového oddílu opět změníme pomocí příkazu `parted`. Při vypsání partition table pomocí příkazu `print` to na nás bude vypisovat varování, že GPT partition table se změnila. V obou případech napište `Fix`. Další úskalí je v tom, že `parted` neumí ve verzi 1.8.1 zvětšovat diskový oddíl typu `xfs`, takže budeme muset tento diskový oddíl zrušit a vytvořit na stejném začátku větší:

    $ parted /dev/sdc

    GNU Parted 1.8.1
    Používám /dev/sdc
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) print                                                           
    Chyba: Záložní tabulka GPT není na konci disku, jak by měla být. To možná znamená, že jiný operační systém si myslí, že disk je menší. Mám to opravit přesunutím zálohy na konec
    (a odstraněním staré zálohy)?
    Opravit/Fix/Ignorovat/Ignore/Zrušit/Cancel? Fix                          
    Varování: Not all of the space available to /dev/sdc appears to be used, you can fix the GPT to use all of the space (an extra 3905945600 blocks) or continue with the current
    setting?
    Opravit/Fix/Ignorovat/Ignore? Fix
    
    Model: DELL PERC H700 (scsi)
    Disk /dev/sdc: 6000GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    
    Number  Start   End     Size    File system  Name  Přepínače
     1      17,4kB  4000GB  4000GB  xfs         1
    
    (parted) unit s                                                          
    (parted) print
    
    Model: DELL PERC H700 (scsi)
    Disk /dev/sdc: 11717836799s
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    
    Number  Start  End          Size        File system  Name  Přepínače
     1      34s 7811891166s  7811891133s  xfs       1            
    
    (parted) rm 1                                                            
    (parted) mkpart 1 34 100%                                                
    (parted) print                                                           
    
    Model: DELL PERC H700 (scsi)
    Disk /dev/sdc: 11717836799s
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    
    Number  Start  End          Size        File system  Name  Přepínače
     1      34s 11717836766s  11717836733s  xfs         1            
    
    (parted) quit                                                            
    Informace: Nezapomeňte aktualizovat /etc/fstab, je-li to potřeba.

Velikost souborového změníme pomocí odpovídajícího příkazu. V případě XFS je to pžíkaz `xfs_grows`, který potřebuje přípojný bod, kde je daný souborový systém připojený, takže je nejprve nutné ho odpojit.

    $ mount /experiment/

A následně ho můžeme zvětšit.

    $ xfs_growfs /experiment

    meta-data =/dev/sdc1 isize=256   agcount=32, agsize=30515199 blks
              =          sectsz=512  attr=1
    data      =          bsize=4096  blocks=976486368, imaxpct=25
              =          sunit=0     swidth=0 blks, unwritten=1
    naming    =version 2 bsize=4096
    log       =internal  bsize=4096  blocks=32768, version=1
              =          sectsz=512  sunit=0 blks, lazy-count=0
    realtime  =none      extsz=4096  blocks=0, rtextents=0
    data blocks changed from 976486368 to 1464729552

Nakonec ověříme, jestli se změna promítla do výpisu příkazu `df`:

    $ df -h

    Filesystem          Size  Used Avail Use% Mounted on
    ...
    /dev/sdc1           5,5T   57G  5,5T   2% /experiment

Ejchuchů! Diskové pole se nám podařilo za běhu zvětšit a možná si toho ani nikdo nevšimnul. Takže si možná nikdo nevšiml, že jste odvedl nějakou práci a možná mají všichni kolem vás pocit, že nejste vůbec potřeba :-(.
