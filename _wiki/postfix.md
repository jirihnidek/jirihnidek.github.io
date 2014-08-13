---
title: Postfix
layout: wiki
tags: linux, mail, network, posftix, smtp
language: Czech
---

> **Poznámka:** Na této stránce naleznete návod jak zkonfigurovat postfix tak, aby se Vám zprávy zasílané různými službami do _/var/spool/mail/_ přeposílaly na Váš osobní e-mail.

Klíčové jsou dva soubory a to _/etc/postfix/main.cf_ a _/etc/aliases_. V _/etc/postfix/main.cf_ by mělo být kromě standardní konfigurace od instalátoru zhruba toto:

    myhostname = nejaky.muj.hostname.cz # adresa daného počítače
    mydomain = hostname.cz
    inet_interfaces = localhost
    relayhost = smtp.hostname.cz # SMTP server
    alias_maps = hash:/etc/aliases
    alias_database = hash:/etc/aliases

No a v _/etc/aliases_ musí být uvedena e-mailová adresa pro všecny uživatele od kterých chcete dostávat poštu:

    root: karel.lopata@domena.cz

To je snad všechno.
