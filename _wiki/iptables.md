---
title: Command iptables
layout: wiki
tags: linux, network, iptables, firewall
language: Czech
---

> **Poznámka:** Tato stránka nemá ambicí být nějakou příručkou k iptables, ale spíše se jedná o moje poznámky a postřehy.

## Výpis pravidel ##

Pro výpis pravidel doporučuji používat následující volby

    iptables -L -v -n --line-numbers
    ip6tables -L -v -n --line-numbers

Význam jednotlivých voleb:

* **-L**: výpis pravidel
* **-v**: podrobný
* **-n**: nepokouší se o preklad adres IP adres
* **--line-numbers**: vypisuje čísla jednotlivých pravidel (důležité, pokud chcete nějaké pravidlo změnit nebo smazat)

## Přidání pravidla ##

Příklad přidání pravidla pro protokol SSH:

    iptables -A INPUT -m state --state NEW -s 147.230.0.0/16 -p tcp --dport 22 -j ACCEPT

Význam jednotlivých voleb a parametrů:

* **-A INPUT**: přidá dané pravidlo na konec seznamu pravidel v chainu příchozích paketů
* **-m state**: použije se modul pro práci se stavama
 * **--state NEW**: musí jít o nový stav
* **-s 147.230.0.0/16**: zdrojová adresa musí být z dané podsítě
* **-p tcp**: povolíme TCP
* **--dport 22**: na portu 22
* **-j ACCEPT**: a předáme paket do akceptovaných paketů

Pro ip6tables je přidání pravidla podobné:

    ip6tables -A INPUT -m state --state NEW -s 2001:718:1c01::/48 -p tcp --dport 22 -j ACCEPT

> **Poznámka:** ne všechny verze ip6tables podporují stavový firewall.

## Drobnosti k SSH ##

Tato sekce se úplně netýká iptables, ale může být důležitá pro zvýšení bezpečnosti tohoto protokolu. Pokud totiž chceme omezit uživatelé, kteří se chtějí přihlásit pomocí SSH pouze na některé lokální uživatele, tak to lze udělat v konfiguračním souboru SSH: _/etc/ssh/sshd_config_ přidáním následujícího řádku:

    AllowUsers josef

Tím SSH serveru říkáme, že má povolit přpojení pouze pro uživatele josef a ostatní uživatele se nebudou moci připojit.

Mnohem flexibilnější je udělat omezení připojování pomocí PAM modulu **pam_access.so**. Jak na to? Do souboru _/etc/pam.d/sshd_ přidáme řádek:

    account    required     pam_access.so accessfile=/etc/ssh/access.conf

kterým PAM subsystému říkáme, že na konci ověřování uživatelského účtu má použít modul pam_access.so, který defaultně používá konfigurační soubor _/etc/security/access.conf_. Pokud chceme použít pro ssh specifický konfigurační soubor, tak to můžeme udělat pomocí volby **accessfile**. Soubor _/etc/ssh/access.conf_ pak může obsahovat podobnou konfiguraci:

    # User josef can login from IPv4 subnet
    +:josef:147.230.0.0/16
    # User josef can login from IPv6 subnet too
    +:josef:2001:718:1c01::/48
    # All other logins of other users are denied
    -:ALL:ALL

Pro bližší informace o struktuře konfiguračního souboru access.conf doporučuji studovat příklady z _/etc/security/access.conf_

## Změna pravidla ##

Ke změně pravidla je dobré znát jeho číslo, proto pro výpis pravidel používám volbu _--line-numbers_. Pokud bychom například chtěli změnit předchozí pravidlo (budeme předpokládat, že jeho číslo je 7), tak to uděláme následujícím způsobem:

    iptables -R INPUT 7 -m state --state NEW -s 147.230.99.0/24 -p tcp --dport 22 -j ACCEPT

Význam jednotlivých voleb a parametrů:

* **-R INPUT 7**: změní pravidlo číslo 7 v chainu příchozích paketů

## Logování ##

Pokud chceme logovat zahozené pakety, tak to lze udělat přidáním příkazu před poslední příkaz (v příkladu předpokládáme, že je na řádku 20), který pakety zahazuje:

    iptables -I INPUT 20 -j LOG --log-level 4
    
    ip6tables -I INPUT 20 -j LOG --log-level 4

Logy se potom budou ukládat do souboru: _/var/log/messages_

## Komentáře ##

Pokud chceme ve výpisu iptables vidět nějaký komentář s tím, co dané pravidlo dělá, tak můžeme použít modul comment:

    iptables -I INPUT 7 -m state --state NEW -p tcp --dport 80 -j ACCEPT -m comment --comment "HTTP"

## Drobnosti pro RedHat/CentOS ##

Linuxová distribuce Red Hat (a z ní distribuce odvozené) májí několik šikovných a užitečných drobností, které mohou člověku zjednodušit život. Nastavení iptables se ukládá do souboru _/etc/sysconfig/iptables_ (IPv4) a _/etc/sysconfig/ip6tables_ (IPv6). Uložení nastavení lze provézt pomocí příkazu:

    /etc/init.d/iptables save
    
    /etc/init.d/ip6tables save

nebo:

    service iptables save
    
    service ip6tables save

Různé nastavení specifické pro RedHat lze nalézt v:

* _/etc/sysconfig/iptables-config_.
* _/etc/sysconfig/ip6tables-config_.

## Změna výchozích pravidel a omezení odchozího spojení ##

Každý _chain_ pravidel má nějaké výzhocí pravidlo. To znamená, že se pakety nejčastěji zahazují nebo propouštějí dál, apod. Pokud například chcete aby výchozípo pravidlo pro odchozí pakety bylo _DROP_, tak je potřeba použít následující příkaz

    iptables -P OUTPUT DROP
    ip6tables -P OUTPUT DROP

Takové pravidlo samozřejmě způsobí, že daný počítač nebude schopen komunikovat po síti. Napravit to můžeme přidáním pravidla:

    iptables -A OUTPUT -j ACCEPT
    ip6tables -A OUTPUT -j ACCEPT

a dále můžeme vybrat služby, které chceme zakázat. Pokud chceme, aby se uživatelé z daného počítače dostali pomocí protkolu HTTP a HTTPS jenom na doménu [www.tul.cz](http://www.tul.cz), tak je potřeba pustit následůcí příkazy:

    iptables -I OUTPUT 1 -p tcp --dport 80 -j DROP
    iptables -I OUTPUT 1 -p tcp --dport 443 -j DROP
    iptables -I OUTPUT 1 -d 147.230.16.27 -p tcp --dport 80 -j ACCEPT
    iptables -I OUTPUT 1 -d 147.230.16.27 -p tcp --dport 443 -j ACCEPT

To samé je vhodné udělat i pro IPv6

    ip6tables -I OUTPUT 1 -p tcp --dport 80 -j DROP
    ip6tables -I OUTPUT 1 -p tcp --dport 443 -j DROP
    ip6tables -I OUTPUT 1 -d 2001:718:1c01:16:216:3eff:fe1a:d23f -p tcp --dport 80 -j ACCEPT
    ip6tables -I OUTPUT 1 -d 2001:718:1c01:16:216:3eff:fe1a:d23f -p tcp --dport 443 -j ACCEPT
