---
title: Command IP
layout: wiki
tags: linux, network, ip
---

Příkaz '''ip''' slouží v Linuxu k zobrazování a správé routovacích tabulek, síťových rozhraní, tunelů a dalších věciček souvisejících se síťováním. Tato stránka nemá ambice být nějakým kompletním manuálem. Slouží jako můj poznámkový blok k danému tématu.

## Vypnutí/zapnutí podpory pro IPv6 ##

Kompletně vypnout podporu IPv6 lze jinak a efektivněji. Pokud člověk potřebuje vypnout podporu pro IPv6 jen na chvíli a pro dané zařízení, tak to lze provést následujícím způsobem:

```bash
 ip -6 addr del <Ipv6 adresa> dev <síťové zařízení>
```

Příklad:

```bash
 ip -6 addr del ::1/128 dev lo
```

Pokud chceme podporu pro IPv6 opět přidat, tak stačí v předchozím příkazu zaměnit klíčové slovo ''del'' za ''add'':

```bash
 ip -6 addr add <Ipv6 adresa> dev <síťové zařízení>
```

Příklad:

```bash
 ip -6 addr add ::1/128 dev lo
```

## Směrování ##

Směrovací tabulku lze vypsat pomocí:

```bash
 $ ip route show
```

a pro IPv6 pomocí:

```bash
 $ ip -6 route show
```

## Nastavení MTU ##

Pokud potřebujete nastavit hodnotu MTU pro nějaké síťové zařízení, tak to lze jednoduše provést pomocí příkazu:

```bash
 $ ip link set <síťové zařízení> mtu <hodnota MTU>
```

Příklad:

```bash
 $ ip link set lo mtu 1500
```

'''Poznámka:''' Defaultní hodnota MTU pro loopback je 16436. Vypsat její hodnotu můžete pomocí příkazu ifconfig:

```bash
 $ ifconfig lo
```

nebo opět pomocí příkazu ip:

```bash
 $ ip link show lo
```
Tohle vše může být užitečné, pokud testujete/ladíte nějakou síťovou aplikaci.
