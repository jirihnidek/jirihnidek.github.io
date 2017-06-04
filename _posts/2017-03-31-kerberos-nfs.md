---
title: Kerberos a NFS (Czech)
layout: post
---

Tento článek vznikl jako uživatelský návod pro uživatele serveru perseus.nti.tul.cz na Technické univerzitě v Liberci nicméně může být i pro další uživatele používající datové úložiště CESNETu. Článek je psaný v češtině protože zde není velká pravděpodobnost výskytu cizojazyčných uživatelů. Jelikož server používá pro připojení datového úložiště NFS, tak návod vychází z oficiální dokumentace CESNETu dostupné na: [https://du.cesnet.cz/cs/navody/nfs/start](https://du.cesnet.cz/cs/navody/nfs/start).

Nejprve je potřeba jako bežný uživatel zkusit vytvořit Kerberos lístek pomocí:

    kinit jmeno_prijmeni@EINFRA

Zde budete požádáni o heslo. Pokud heslo nemáte, tak je potřeba se zaregistrovat na: [https://einfra.cesnet.cz/](https://einfra.cesnet.cz/)

> Poznámka: uživatelské jméno nemusí být nutně ve formátu: `jmeno_prijmeni`, ale záleží na tom jaké jste si zvolili uživatelské jméno při registraci na stránkách datového úložiště.

Po úspěšném zadání hesla byste měli být schopni vypsat lístek pomocí:

    klist -l

Výstupem by mělo být něco jako:

    [jiri.hnidek@perseus ~]$ klist -l
    Principal name                 Cache name
    --------------                 ----------
    jmeno_primeni@EINFRA             FILE:/tmp/krb5cc_1000

Vytvořte si ve svém domovském adresáři prázdný adresář například pomocí:

    mkdir cesnet-du3

Následně požádejte správce systému, aby Vám vytvořil v souboru `/etc/fstab` podobný záznam:

    nfs.du3.cesnet.cz:/	/home/jmeno.prijmeni/cesnet-du3   nfs4   sec=krb5,rsize=1048576,wsize=1048576,user   0 0

Správce systému Vám musí vytvořit i záznam v souboru `/etc/idmap.conf`, kde je nastavené mapování vašeho lokálního uživatelského jména na uživatelské jméno na datovém úložišti CESNETu. V tomto souboru v sekci `[Static]` by se mělo objevit něco jako:

    nfs/jmeno_prijmeni@EINFRA = jmeno.prijmeni

Takže správci prosím sdělte obě Vaše uživatelská jména. Trochu mu tím ušetříte práci.

Po vytvoření Kerberos lístku a záznamu ve `/etc/fstab` je možné si připojit NFS "disk" datového úložiště pomocí následujícího příkazu:

    mount /home/jmeno.prijmeni/cesnet-du3

Teď můžete začít využívat datové úložiště, ale POZOR všechny data ukládejte do adresáře:

    /home/jmeno.prijmeni/cesnet-du3/home/jmeno_prijmeni/VO_tul_fm_nti-tape_tape

Pokud data budete ukládat pouze do `/home/jmeno.prijmeni/cesnet-du3/home/jmeno_prijmeni`, tak vaše data nebudou na datovém úložišti zálohovaná, nebud přístupná z rychlých disků, budete mít dostupnou omezenou diskovou kapacitu a vůbec to budete dělat špatně, takže Vám administrátor datového úložiště brzo pošle výhružný e-mail.

## Připojování NFS svazku bez nutnosti zadávat heslo

Abyste nemuseli vždy při pokusu o mapování disků na vizualizační server zadávat heslo, tak je vhodné si vytvořit tzv. keytab soubor pomocí:

    ssh -o PubkeyAuthentication=no -o GSSAPIAuthentication=no jmeno_prijmeni@nfs.du3.cesnet.cz "remctl kdc1.cesnet.cz accounts nfskeytab" > ~/krb5.keytab

Pokud jste zadali správně své uživatelské jméno, tak by se vám měl ve vašem domovském adresáři vytvořit soubor `krb5.keytab`. Toto je bohužel akce, kterou nelze nijak automatizovat, protože nikdo za Vás nemůže zadat uživatelské heslo, které znáte pouze vy osobně. O existenci tohoto souboru informujte správce vizualizačního serveru. Soubor není nikterak nutné posílat e-mailem. Stačí stručná informace, že jste vytvořili soubor krb5.keytab ve vašem domovském adresáři.

Tento soubor administrátor severu zkopíruje do adresáře `/etc/krb5.keytab.d/jmeno_prijmeni.keytab` a v souboru `/etc/cron.daily/krb5_kinit.cron` přidá příslušný záznam pro nový keytab soubor. Tento soubor je následně používán pro pravidelnou aktualizaci Vašeho kerberos lístku.
