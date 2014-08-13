---
title: Linux and Libraries
layout: wiki
tags: linux, libraries
language: Czech
---

> **Poznámka:** toto jsou moje osobní poznámky o chování a vytáření různých knohoven pod OS Linux

Knihovny a systémové proměnné
-----------------------------

Pokud potřebujete otestovat nějakou sdílenou knihovnu, tak není potřebovat instalovat ji někam do systému. Postačí, když vytvoříte systémovou proměnnou '''LD_LIBRARY_PATH''' a nastavíte ji příznak pro vyexportování.

```bash
export LD_LIBRARY_PATH=/path/to/your/library
```

Pokud potřebujete testovat knihovnu s Pythoním modulem, tak na to potřebujete nastavit jinou systémovou proměnnou '''PYTHONPATH'''

```bash
$ export PYTHONPATH=/path/to/your/python/module
```

Externí odkazy
--------------

* [TLDP Shared Libraries](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)
