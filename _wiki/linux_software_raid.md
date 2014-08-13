---
title: Linux Software RAID
layout: wiki
tags: linux, raid
language: Czech
---

Tyto stránky slouží jako moje poznámky k Linuxovému nástroji **mdadm**, který slouží k vytváření a správě softwarového RAIDu pod OS Linux.

## Vytvoření softwarového RAIDu ##

Než začneme vytvářet softwarový radi, tak je potřeba rozdělit potřebné disky (alespoň dva) na několik (alespoň jeden :-)) diskových oddílů. Tyto diskové oddíly musí být označeny jako *Linux RAID samorozpoznatelný*. Rozdělení disků provedeme pomocí nástroje **fdisk** nebo **parted**.

> Poznámka: softwarový RAID lze testovat i na jednom disku, kdy si disk rozdělíme na několik diskových oddílů. Takové použití ovšem nemá žádný praktický význam, protože při poškození tohoto disku přijdeme o všechna data. RAID má obecně sloužit jako prevence proti ztrátě dat při selhání jednoho z disků.

Předpokládejme, že máme vytvořeno na 4 discích 4 diskové oddíly, které jsou označené jako *Linux RAID samorozpoznatelný*. Tyto diskové oddíly mají odpovídající blokové zařízení: */dev/sda2*, */dev/sdb2*, */dev/sdc2* a */dev/sdd2*. S takovým počtem disků si můžeme vytvořit například RAID5 pomocí následujícího příkazu:

    mdadm --create /dev/md0 --level=5 --raid-devices=4 /dev/sda2 /dev/sdb2 /dev/sdc2 /dev/sdd2

Parametrem */dev/md0* říkáme, že */dev/md0* bude blokové zařízení, které se bude používat k práci s vlastním RAIDem. Volbou *--level=5* říkáme programu mdadm, že chceme vytvořit raid5. Volba *--raid-devices=4* říká, že RAID 5 bude obsahovat 4 zařízení. Nakonec předáme programu 4 parametry s příslušnými blokovými zařízeními, které budou tvořit vlastní RAID.

Příkazem

    mdadm -D /dev/md0

můžeme zjist v jakém stavu se RAID nachází. Výpis by měl obsahovat něco podobného:

    /dev/md0:
            Version : 00.90.03
      Creation Time : Fri Oct 24 13:02:27 2008
         Raid Level : raid5
         Array Size : 312960 (305.68 MiB 320.47 MB)
      Used Dev Size : 104320 (101.89 MiB 106.82 MB)
       Raid Devices : 4
      Total Devices : 4
    Preferred Minor : 0
        Persistence : Superblock is persistent
    
        Update Time : Fri Oct 24 13:02:38 2008
              State : clean, degraded, recovering
     Active Devices : 3
    Working Devices : 4
     Failed Devices : 0
      Spare Devices : 1
    
             Layout : left-symmetric
         Chunk Size : 64K
    
     Rebuild Status : 55% complete
    
               UUID : 003b8b54:2fb3c710:4f0683dc:de30543b (local to host blender)
             Events : 0.4
    
        Number   Major   Minor   RaidDevice State
           0       8       17        0      active sync   /dev/sda2
           1       8       18        1      active sync   /dev/sdb2
           2       8       19        2      active sync   /dev/sdc2
           3       0        0        -      removed
    
           4       8       21        3      spare rebuilding   /dev/sdd2

Co to znamená, že po vytvoření RAIDU se jeden diskový oddíl označí jako spare a začne jeho obnova ze tří prvních disků. Než se RAID dostane do "čistého" stavu, tak může chvilku trvat v závislosti na velikosti a rychlosti vašich diskových oddílu. Vlastní zformátování diskového oddílu a používání bych provedl až, když je RAID v "čistém" stavu a vaše data jsou chráněna.

Při vytváření RAIDU je možné raid5 vytvořit v tzv. degradovaném stavu, kdy mu nedáte k dispozici všechny potřebné diskové oddíly:

    mdadm --create /dev/md0 --level=5 --raid-devices=4 /dev/sda2 missing /dev/sdc2 /dev/sdd2

Jaké to má výhody a nevýhody? Nevýhoda je celkem jasná. vaše data nejsou tolik chráněna, protože při výpadku jednoho z disků přijdete o vše! Výhoda je v kombinaci s paramatrem **--assume-clean**, kdy vytváříme RAID již z diskových oddílů, které již byly součástí nějakého RAIDu (toto se velmi hodí, pokud řešíte nějaké problémy).

## Přidání Spare disku ##

Pokud máme k dispozici další disk, tak ho můžeme přidat jako tzv. spare disk. Takový disk slouží jako záložní disk. Když se jeden z disků poškodí, tak je tento disk použit okamžitě k obnově RAIDU. Disk musí mít stejné rozdělení jako předchozí disky. Jeho přidání provedeme následujícím příkazem:

    mdadm /dev/md0 -a /dev/sde2

Do RAIDU můžeme přidat libovolné množství spare disků, ale to je plýtvání. Spíše se využívá toho, že jeden spare disk je sdílen mezi více RAIDy. Výpis pomocí:

    mdadm -D /dev/md0

po přidání spare disku vypadá následovně:

    /dev/md0:
            Version : 00.90.03
      Creation Time : Fri Oct 24 13:02:27 2008
         Raid Level : raid5
         Array Size : 312960 (305.68 MiB 320.47 MB)
      Used Dev Size : 104320 (101.89 MiB 106.82 MB)
       Raid Devices : 4
      Total Devices : 5
    Preferred Minor : 0
        Persistence : Superblock is persistent
    
        Update Time : Fri Oct 24 13:07:27 2008
              State : clean
     Active Devices : 4
    Working Devices : 5
     Failed Devices : 0
      Spare Devices : 1
    
             Layout : left-symmetric
         Chunk Size : 64K
    
               UUID : 003b8b54:2fb3c710:4f0683dc:de30543b (local to host blender)
             Events : 0.6
    
        Number   Major   Minor   RaidDevice State
           0       8       17        0      active sync   /dev/sda2
           1       8       18        1      active sync   /dev/sdb2
           2       8       19        2      active sync   /dev/sdc2
           3       8       21        3      active sync   /dev/sdd2
    
           4       8       22        -      spare   /dev/sde2

## Odstranění vadného disku ##

Pokud chceme odstranit vadný disk, tak pokud tomu tak již není, tak je potřeba ho za vadný označit:

    hdadm /dev/md0 -f /dev/sdb2

pak je potřeba vadný disk odebrat:

    hdadm /dev/md0 -r /dev/sdb2

V této situaci se buď diskové pole nachází v degradovaném stavu (pokud jste neměli přidaný spare disk), nebo začala obnova dat na spare disk.
