---
title: Traffic Control
layout: wiki
---

Traffic Control je nástroj pro prioritizaci paketů v operačním systému Linux. Nastavení se provádí pomocí příkazu *tc*. Může sloužit pro zlepšení propustnosti sítě případně pro simulování některých stavů sítě (packet loss, delay, atd.)

Pozn.: toto jsou pouze moje osobní poznámky, ve kterých se můžou vyskytovat nepřesnosti, podivná vyjádření nebo naprosté bludy.

## Základní pojmy ##

### Queues (fronty) ###

Každý paket, který má být odeslán, je zařazen od tzv. fronty (queue). Na frontu můžeme pohlížet jako na paměť typu FIFO, která ovšem není nekonečně velká. Pokud se paket do fronty nevejde, tak je zahozen.

                  +------+-+-+-+
    ---packet-->--+      | | | +---->
                  +------+-+-+-+

Defaultně má každé síťové rozhraní (NIC) jednu frontu. S jednou frontou toho moc nepořídíme, takže Traffic Control se provádí pomocí více front. Každá fronta pak může mít různou prioritu, zpoždění, ztrátovost, apod. Každá fronta může navíc obsahovat další fronty.

### Flows (toky) ###

Toky jsou pakety se stejnou IP adresou odesílatele i příjemce a stejnými porty.

### QoS (Quality of Services) ###

Pokud někdo nebo něco poskytuje QoS, tak musí být schopný zaručit takové věci jako je šířka pásma, latence, ztrátovost, atd., ale ani v tom případě nemáte zaručené tyto parametry 100%. Tyto parametry jsou garantované s nějakou předem dohodnutou pravděpodobností (např. na 99,99%).

### Tokeny ###

Tento token nemá nic společného s Token Ringem. Jde o to, že TC se lépe provádí s většími shluky paketů než se samostatnými pakety. Tyto shluky paketů se vyřizují v pravidelných časových intervalech a říká se jim tokeny. Můžeme si to přirovnat k lidem, kteří čekají ve frontě na horskou dráhu. Z této fronty do vláčku nastoupí v pravidelném intervalu pouze určitý počet lidí.

### Bukety ###

Buket si můžeme z předchozí analogie vysvětlit jako maximální počet volných vláčků, které mohou čekat u fronty na náhlý nával zákazníků.

### Burst Traffic ###

Burst traffic můžeme vysvětlit jako návalový traffic a v naší analogii si to můžeme ukázat na situaci, kdy do zábavního parku přijede několika autobusy velké množstvím turistů a všichni se najednou vrhnou na horskou dráhu. Tento stav často odpovídá přenosu pomocí protokolu HTTP.

### Shaping ###

Shaping můžeme s trochou fantazie přeložit jako tvarování. Jeho hlavní úlohou je "vyhlazovat" burst traffic. Síťový prvek, kde se provádí shaping zpožďuje pakety takovým způsobem, aby dosáhl požadované rychlosti.

**Non-work-conserving mechanismus**: přestože ve výstupní frontě jsou pakety, které by se mohli poslat, tak se nepošlou.

**Work-conserving mechanismus**: pakety ve výstupní frontě se posílají vždy, když je to možné.

U shapingu se uplatňuje právě non-work-conserving mechanismus.

### Sheduling ###

Sheduling je mechanismus, kdy dochází ke změně pořadí paketů na výstupní frontě. Správný sheduler by měl splňovat 4 základní pravidla:

* **Jednoduchá implementace**: pro *N* spojení by operační náročnost měla být *O(1)*. Operační náročnost třeba i *O(n)* je nepřijatelná, protože směrovače často obhospodařují desetitisíce až statisíce spojení a jakékoliv spoždění na směrovači je kritické.
* **Férovost a ochrana negarantovaných služeb**: férovost znamená, že systémové zdroje jsou rovnoměrně rozděleny mezi všechny spojení (uvažujeme stejnou prioritu všech spojení). Ochrana služeb znamená, že zbabělé a neférové chování jedneho spojení bude umravněno a negativně neovlivní ostatní spojení.
* **Garantovaná kvalita spojení**: parametry, které se většinou garantují: šířka pásma (bandwidth), spoždění (delay), rozkmitání spoždění (delay jitter) a ztrátovost paketů (loss).
* **Jednoduchá a efektivní přístupová kontrola**: rozhodování o tom, jestli bude nové spojení přijato, tak aby nebyly ohroženy existující spojení. Zároveň musí být zajištěno, aby nebyly nová spoejení odmítána zbytečně a nedocházelo k neefektivnímu využívání sítě.

### Klasifikace ###

Třídění a přiřazování trafiku (paketů) do různých front.

### Policing (dohled) ###

Dohled na trafikem v jednotlivých frontách.

### Droping (zahazování) ###

K zahazování paketů, toků dochází tehdy, když dojdou systémové zdroje :-)

### Značkování ###

Mechanismus, kdy dojde ke změně paketu.

## Qdisc (Queue discipline) ##

**qdisc**, neboli queue discipline (český pokus o překlad: disciplína ve frontách) je stručně a jednoduše sheduler, který hází pakety do jednotlivých front. qdisc můžeme nastavovat pro odchozí pakety *root* (většina případů) nebo pro příchozí pakety *ingress*, což je většinou zbytečné, protože když už k nám paket jednou dorazil, tak jsme tomu rádi a nebudeme ho strkat ještě do nějaké fronty.

qdisc nastavujeme pomocí příkazu **tc**. Kompletní disciplínu vytváříme spojování jednotlivých bloků qdisc.

qdisc dělíme na **clasful** a **classless**. Každé síťové zařízení má defaultně přiřazen jeden classless qdisc, který se chová jako FIFO. Každý classful qdisc může obsahovat další libovolný počet qdisců, které se mohou dále větvit. Celé si to můžeme představit jako stromovou strukturu, kde koncové uzlu se nazývají listy a obsahují FIFO qdisc.

Classful qdisc jsou obecně shedulery a classless qdis jsou obecně shapery.

### Praktické příklady ###

Pro hraní si s *tc* je podle mého skromného názoru nejlepší si nejprve zprovoznit nějaký virtuální stroj, například pomocí [KVM](http://kvm.qumranet.com/kvmwiki) (nebo [VmWare Player](http://www.vmware.com/products/player/), [Xen](http://www.xen.org/), [VirtualBox](http://www.virtualbox.org/), apod.). Když si pomocí *tc* nastavujete parametry pro virtuální síťové rozhraní, tak neovlivňujete vlastnosti síťového zařízení, které používáte pro komunikaci s okolním světem a která pro vás může být důležitá. Navíc takto můžete jednoduše testovat vliv vaší změny na virtuální síťové rozhraní.

Příklad virtuální síťové zařízení na hypervizorovi:

    ifconfig vnet0

    vnet0     Link encap:Ethernet  HWadr 00:ff:b1:78:7c:6e  
              inet adr:192.168.122.1  Všesměr:192.168.122.255  Maska:255.255.255.0
              inet6-adr: fe80::a4d8:84ff:fef8:5175/64 Rozsah:Linka
              AKTIVOVÁNO VŠESMĚROVÉ_VYSÍLÁNÍ BĚŽÍ MULTICAST  MTU:1500  Metrika:1
              RX packets:7 errors:0 dropped:0 overruns:0 frame:0
              TX packets:164 errors:0 dropped:0 overruns:0 carrier:0
              kolizí:0 délka odchozí fronty:0 
              Přijato bajtů: 456 (456.0 B) Odesláno bajtů: 37479 (36.6 KB)

Síťové zařízení na uzlu:


    $ ifconfig eth0


    eth0      Link encap:Ethernet  HWaddr 52:54:00:95:15:95  
              inet addr:192.168.122.12  Bcast:192.168.122.255  Mask:255.255.255.0
              inet6 addr: fe80::5054:ff:fe95:1595/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:7 errors:0 dropped:0 overruns:0 frame:0
              TX packets:25 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000 
              RX bytes:570 (570.0 B)  TX bytes:1742 (1.7 KB)
              Interrupt:11 Base address:0xc100

Následující pokusy budeme provádět na hostu pro síťové rozhraní eth0.

#### Jak jsme na tom? ####

Když chceme vytisknout stav qdics pro nějaké síťové rozhraní, tak použijeme následující příklad:

    $ tc -s -d qdisc show dev eth0

Pro defaultně nastavené síťové rozhraní dostaneme následující výstup:

    qdisc pfifo_fast 0: root bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
     Sent 55652 bytes 683 pkt (dropped 0, overlimits 0 requeues 0) 
     rate 0bit 0pps backlog 0b 0p requeues 0

Tento výpis nám říká, že máme jednu qdisc, která je typu **pfifo_fast**. Tato qdisc se používá pro pakety, které jdou ven (parametr root). To, že je qdisc typu pfifo_fast znamená, že se jedná o tři roury (bands 3). Každá roura je označena číslem od 0 do 2. Pokud se v rouře 0 nachází nějaký paket, tak má přednost před pakety v rourách 1 a 2. Podobně, pokud je v rouře 1 nějaký paket, tak má přednost před pakety v rouře 2. **priormap** je prioritní mapa, která nám říká, jak jsou jednotlivé [IP](http://en.wikipedia.org/wiki/IPv4) pakety řazeny do jednotlivých rour podle [TOS](http://en.wikipedia.org/wiki/Type_of_Service). Díky tomu se pakety s větší prioritou dostanou ven dříve jak ostatní.

Délku roury získáme použitím příkazu **ip**:

    $ ip link list

* vypíše seznam zařízení a informace o nich (stav, typ, MTU, MAC, qdisc). Nás bude zajímat, že nás informuje o délce pfifo_fast.

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue 
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
        link/ether 52:54:00:95:15:95 brd ff:ff:ff:ff:ff:ff

V případě rozhraní eth0 je jeho délka 1000. Tento parametr lze nastavit pouze pomocí příkazu **ip** (případně pomocí ifconfig). Tento parametr nelze nastavit pomocí příkazu tc.

    $ ip link set dev eth0 txqueuelen 100

Pomocí příkazu ip lze zobrazovat nebo měnit spoustu dalších věcí:

    $ ip address show

* vypíše navíc IP adresy

    $ ip route show

* vypíše routovací tabulky

    $ ip neigh show

* vypíše ARP cache

## Syntaxe příkazu tc ##

Příkaz *tc* tedy umožňuje zobrazovat nastavení pro traffic control:

    $ tc [-s|-d] qdisc show [dev DEV]

    $ tc [-s|-d] class show dev DEV

    $ tc filter show dev DEV

nebo měnit nastavení:

    $ tc qdisc [add|change|replace|link] dev DEV [parent qdisc-id|root] \
      [handle qdisc-id] qdisc [qdisc specific parameters]

    $ tc class [add|change|replace] dev DEV parent qdisc-id \
      [classid class-id] qdisc [qdisc specific parameters]

    $ tc filter [add|change|replace] dev DEV [parent qdisc-id|root] \
      protocol protocol prio priority filtertype [filtertype specific parameters] flowid flow-id

## Queue discipline Netem ##

Jednou z možných qdisc je netem, který umožňuje simulování parametrů skutečné linky. Toto se například hodí, pokud vyvíjíme nový síťový protokol.

### Simulace zpoždění síťových paketů ###

Zpoždění na síťovém rozhraní lze nastavit pomocí příkazu *tc*. Než si ukážeme, jak zpoždění nastavit, tak si zkusíme pingnout z virtuálního stroje na hypervizora:

    $ ping 192.168.2.1

Výstupem bude něco takového:

    64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=0.041 ms
    64 bytes from 192.168.2.1: icmp_seq=2 ttl=64 time=0.033 ms
    64 bytes from 192.168.2.1: icmp_seq=3 ttl=64 time=0.039 ms
    64 bytes from 192.168.2.1: icmp_seq=4 ttl=64 time=0.037 ms

Z výpisu můžeme vidět, že zpoždění je v průměru menší jak desetina milisekundy, což příliš neodpovídá reálnému síťovému provozu (desítky až stovky milisekund).

Zpoždění můžeme nastavit pomocí následujícího příkazu:

    $ tc qdisc add dev eth0 parent root handle 1:0 netem delay 100ms

kde se příkazu *tc* říká, aby přidal *qdisc* (qdisc add) pro NIC eth0 (dev eth0), jeho rodičem bude qdisc root (parent root), jeho jednoznačný identifikátor bude 1:0 (handle 1:0) a vlastní qdisc bude *netem*, který bude pakety zpožďovat o 100 ms.

Nyní můžeme zkusit příkaz ping a měli bychom dostat podobný výsledek:

    64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=191 ms
    64 bytes from 192.168.2.1: icmp_seq=2 ttl=64 time=104 ms
    64 bytes from 192.168.2.1: icmp_seq=3 ttl=64 time=104 ms
    64 bytes from 192.168.2.1: icmp_seq=4 ttl=64 time=104 ms

Můžeme vidět, že zpoždění paketů se v průměru zvýšilo o více jak 100ms. Pokud bychom chtěli mít symetrické a realitě podobnější zpoždění, tak je potřeba nastavit stejné zpoždění i na opačné straně (na hypervizorovi):

    $ tc qdisc add dev vnet0 parent root handle 1:0 netem delay 100ms

Pokud bychom chtěli získat ještě realističtější charakter zpoždění, který nebude víceméně konstantní, tak lze příkaz vylepšit o rozptyl zpoždění a korelaci na předchozí hodnotě zpoždění:

    $ tc qdisc change dev eth0 parent root handle 1:0 netem delay 100ms 10ms 25%

Jinými slovy tímto nastavujeme tzv. **delay jitter** (česky roztřesení zpoždění), které je kritické u přenosu streamovaných médií. Každý přehrávač (videa, nebo zvuku), který přehrává streamované médium si toto médium ukládá do bufferu, aby vyrovnal krátkodobé zpoždění přenosu. Pokud je tento výpadek větší než odpovídá velikosti vyrovnávacího bufferu, tak dojde ke krátkému přerušení vysílání.

Na výstup příkazu ping můžeme vidět, že zpoždění osciluje kolem hodnoty 100ms:

    64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=192 ms
    64 bytes from 192.168.2.1: icmp_seq=2 ttl=64 time=108 ms
    64 bytes from 192.168.2.1: icmp_seq=3 ttl=64 time=104 ms
    64 bytes from 192.168.2.1: icmp_seq=4 ttl=64 time=92.3 ms

Pokud chcete získat zpoždění s normálním rozdělením, tak předchozí příkaz musíme trochu upravit:

    $ tc qdisc change dev eth0 parent root handle 1:0 netem delay 100ms 20ms distribution normal

Pokud chcete mít nějaké úplně jiné rozdělení chyb, tak i to je možné. Pro více detailů nahlédněte do adresáře: */usr/lib/tc* kde jsou textové "konfigurační" soubory pro jednotlivá rozdělení.

### Smazání qdisc ###

Když už simulaci zpoždění ztráty paketů nepotřebujeme, tak smažeme příslušný *qdisc* pomocí následujícího příkazu:

    $ tc qdisc del dev eth0 parent root handle 1:0 netem

na konci musí být *qdisc*, který je nastavený pro daný handle (nemusí to být pouze netem, ale třeba pfifo, htb, apod.). Po odstranění posledního *qdisc* se nastavení traffic controlu vrátí do výchozího stavu, který byl popsán na začátku.

### Simulace ztráty paketů ###

Pokud chceme simulovat ztrátu paketů, tak na to opět použijeme příkaz *tc*:

    $ tc qdisc change dev eth0 root netem loss 20%

Tímto příkazem jsme nastavili paket loss na 20%. Je nutné si uvědomit, že paket loss se nám efektivně projeví u protokolů, které neposkytují spolehlivé datové spojení jako je UDP, ICMP, IGMP, DCCP, RTP, apod. Neprojeví se například u TCP, HTTP, apod.

Paket loss můžeme nejjednodušeji vyzkoušet opět pomocí příkazu ping (používá protokol ICMP). Výstup potom bude následující

    64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=212 ms
    64 bytes from 192.168.2.1: icmp_seq=2 ttl=64 time=111 ms
    64 bytes from 192.168.2.1: icmp_seq=3 ttl=64 time=96.3 ms
    64 bytes from 192.168.2.1: icmp_seq=4 ttl=64 time=83.0 ms
    64 bytes from 192.168.2.1: icmp_seq=6 ttl=64 time=74.4 ms
    64 bytes from 192.168.2.1: icmp_seq=10 ttl=64 time=85.8 ms
    64 bytes from 192.168.2.1: icmp_seq=11 ttl=64 time=125 ms
    
    --- 192.168.122.12 ping statistics ---
    11 packets transmitted, 7 received, 36% packet loss, time 10009ms
    rtt min/avg/max/mdev = 74.491/112.693/212.474/43.811 ms

Z výpisu můžeme vidět, že některé pakety se skutečně ztratili a jaká byla výsledná ztrátovost. Ta se v tomto případě liší od nastavené, protože bylo odesláno malé množstv paketů. Už kolem 100 odeslaných paketech by se naměřená hodnota začala více blížit nastavené hodnotě.

Je nutné si uvědomit, že když nastavíme ztrátovost i na opačné straně spojení, tak se budou výsledné ztrátovosti sčítat.

Pokud bychom chtěli získat přirozenější chování ztrátovosti, tak můžeme předchozí příklad opět trochu modifikovat:

    $ tc qdisc change dev eth0 root netem loss 20% 25%

### Simulace duplikace paketů ###

Pokud chceme simulovat duplikaci paketů, tak nám na to může posloužit volba *duplicate* a parametr určující kolik procent paketů bude duplikováno:

    $ tc qdisc change dev eth0 parent root handle 1:0 netem duplicate 1%

### Simulace poškození paketů ###

Pro simulaci poškození paketů slouží volba *corrupt* a parametr, který opět určuje kolik procent bitů bude poškozeno:

    $ tc qdisc change dev eth0 parent root handle 1:0 netem corrupt 0.1% 

Tato volba nemá příliš smysl, pokud testujeme protokoly, které v sobě integrují kontrolní součty (např. UDP, TCP, apod.).

### Simulace prohození pořadí paketů ###

Simulovat prohození paketů lze různými způsoby. Jeden z nich lze provést takto:

    $ tc qdisc change dev eth0 parent root handle 1:0 netem delay 10ms reorder 25% 50%

### Kombinace jednotlivých parametrů ###

*Qdisc* samozřejmě umožňuje kombinovat jednotlivé parametry, tak abychom docílili realistických a hlavně požadovaných vlastností linky. Příklad:

    $ tc qdisc change dev eth0 parent root handle 1:0 \
    netem delay 100ms 20ms distribution normal loss 5% duplicate 1%

## Seznam zdrojů a literatury ##

* [Linux Advanced Routing & Traffic Control](http://lartc.org/)
* [Traffic Control HOWTO](http://tldp.org/HOWTO/Traffic-Control-HOWTO/index.html)
* [Practical Guide to Linux Traffic Control](http://trekweb.com/~jasonb/articles/traffic_shaping/index.html)
* [An Engineering Approach to Computer Networking](http://www.amazon.com/exec/obidos/tg/detail/-/0201634422/104-4070629-3069516?v=glance)
* [The Linux Foundation: Netem](http://www.linuxfoundation.org/collaborate/workgroups/networking/netem)
