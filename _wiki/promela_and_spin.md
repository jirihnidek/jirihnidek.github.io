---
title: Promela and Spin
layout: wiki
tags: network, promela, spin
language: Czech
---

Tato stránka obsahuje moje osobní poznámky o programovacím jazyku Promela a nástroji Spin. Spin může sloužit nejen k simulaci modelu, který byl vytvořen v Promele, ale může sloužit i k verifikaci systému popsaného pomocí tohoto modelu. Tato stránka neslouží jako dokumentace k programovacímu jazyku Promela, ale poskytuje jednoduché návody a postupy pro konkrétní příklady.

Na této stránce jsou jak příklady naprosto hloupých systémů, které obsahují nějakou chybu a zároveň se zde nacházejí systémy bez vad. U každého systému je zmíněno o jaký systém se jedná.

## Promela, WTF?! ##

Když člověk poprvé uvidí nějaký zdrojový soubor napsaný k jazyku Promela, tak se mu pravděpodobně protočí panenky a možná prohodí i nějakou méně slušnou nadávku. Když se bavíme o Promele, tak je důležité si uvědomit, že tento jazyk nebyl navržen na programování aplikací, které používají lidské bytosti. Tento jazyk byl vytvořen pro ověření správnosti návrhu paralelních systémů. Cože? Ověření správnosti návrhu paralelního systému? Mluvte na mě česky! Dobrá, zkuste si představit, že navrhujete například nějakou klient-server aplikaci. Tomu asi člověk rozumí. Máme nějakého klienta (například webovou aplikaci ve webovém prohlížeči) a nějaký server (nějakou PHP aplikaci běžící na webovém serveru). To asi každý zná. No jo, ale na tom kurňa není nic složitého, nebo ne? Právě, že to může být velmi složité. Jak klient, tak server se mohou nacházet v různých stavech a v každém stavu se předpokládá, že protostrana bude posílat nějaké zprávy. No a co když je nebude posílat, vypadne spojení, server spadne, dojde k invazi masožrvých slimáků, atd. S tím vším se musí při návhru aplikace, respektivě komunikačního protokolu počítat a Promela se Spinem nám mohou ověřit, že jsme na něco nazapomněli.

## Stavy a stavový automat ##

V Promele se nám bude Programovat mnohem snáze, pokud si člověk nejprve vezme tužku a papír a nakreslí si na ně stavy a stavový diagram(y) všech různých entit (klient, server, výpočetní uzel), které spolu budou komunikovat. Člověku už během kreslení stavového diagramu dojde spoustu věcí, uvědomí si, kde všude musí počítat s timeouty, apod.

## Detekce uváznutí ##

Příkladem uváznutí nebo-li deadlocku je uveden na následujícím příkladu:

    proctype Client(chan in; chan out)
    {
      byte i;
         
      do
      :: in?i ->
        if
        :: (i>0) -> { goto end; };
        :: else -> { out!i; };
        fi;
      od;
     
      end:
        skip;
    }
    
    proctype Server(chan in; chan out)
    {
      byte i;
    
      do
      :: in?i -> atomic {
        out!i+1;
        goto end;
        }
      od;
    
      end:
        skip;
    }
    
    init
    {
      chan q1 = [0] of {byte};
      chan q2 = [0] of {byte};
     
      atomic{run Server(q1, q2); run Client(q2, q1);};
    }

Tento [deadlock.prm](/data/deadlock.prm) lze simulovat jednoduše pomocí příkazu:

```bash
spin deadlock.prm
```

Výstupem takové simulace bude následující výstup:

          timeout
    #processes: 3
      2: proc  2 (Client) line   5 "deadlock.prm" (state 10)
      2: proc  1 (Server) line  21 "deadlock.prm" (state 5)
      2: proc  0 (:init:) line  38 "deadlock.prm" (state 4) <valid end state>
    3 processes created

Který nám říká, že pouze počáteční proces ''init'' skončil v korektním koncovém stavu a ostatní procesy skončily v nějakém deadlocku. Abychom zjistili, kde došlo k deadlocku, tak je potřeba z modelu v jazyce Promela vytvořit verifikator a to uděláme pomocí následujících příkazů:

```bash
spin -a deadlock.prm
```

,který nám nejprve vygeneruje zdrojový soubor pan.c v jazyku C (plus další pomocné soubory). Zdrojový soubor pan.c přeložíme pomocí:

```bash
gcc -o deadlock pan.c
```

Výsledný verifikátor ''deadlock'' spustíme pomocí:

```bash
./deadlock
```

jehož výstup má podobu:

    hint: this search is more efficient if pan.c is compiled -DSAFETY
    pan:1: invalid end state (at depth 1)
    pan: wrote deadlock.prm.trail
    
    (Spin Version 5.2.5 -- 17 April 2010)
    Warning: Search not completed
     + Partial Order Reduction
    
    Full statespace search for:
     never claim           - (none specified)
     assertion violations  +
     acceptance   cycles   - (not selected)
     invalid end states  +
    
    State-vector 36 byte, depth reached 2, errors: 1
            2 states, stored
            0 states, matched
            2 transitions (= stored+matched)
            1 atomic steps
    hash conflicts:         0 (resolved)
    
        2.501  memory usage (Mbyte)
    
    pan: elapsed time 0 seconds

Abychom zjistili, kde je chyba, tak je potřeba vygenerovaný soubor ''deadlock.prm.trail'' předhodit spinu:

```bash
spin -k deadlock.prm.trail deadlock.prm
```

,který nám řekne, že chyba nastala přesně tady:

    spin: trail ends after 2 steps
    #processes: 3
      2: proc  2 (Client) line   5 "deadlock.prm" (state 10)
      2: proc  1 (Server) line  21 "deadlock.prm" (state 5)
      2: proc  0 (:init:) line  38 "deadlock.prm" (state 4) <valid end state>
    3 processes created

Což, je vlastně totožný výstup jako jsme dostali při simulaci ;-). Omlouvám se, jestli jste došli až sem, ale tento přístup je užitečný pokud je systém komplikovaný a k deadlocku dochází za velmi speciálních případů, které je potřeba odhalit. Výstup nám říká, že proces Client se zasekl na řádku 5 a proces Server se zaseknul na řádku 21. Důvodem je to, že oba procesy čekají na příjem zprávy, ale nikdo z nich nepošle první počáteční zprávu. Pokud bychom na 4. řádek umístili příkaz ''out!0'', tak k tomuto deadlocku nedojde.

## Detekce nekorektních cyklů ##

Přestože v systému nedojde k uváznutí a na první pohled se může zdát, že se se systémem něco děje, tak se systém může nacházet ve stavu, který není korektní. Obecně lze říci, že se v systému vyskytují nepovolené cykly, které rozlišujeme na nevyvíjející se cykly a livelocky.

### Detekce nevyvíjejících se cyklů ###

TODO

### Detekce livelocků ###

TODO

## Reference ##

* [spinroot.com](http://spinroot.com)
* [Spin a Promela](http://www.fi.muni.cz/~xbarnat/IV101/IV101-03-SPIN.pdf)
* [Logic Model Checking](http://www.cs.utep.edu/sroach/F07-5383/lecture07.pdf)
