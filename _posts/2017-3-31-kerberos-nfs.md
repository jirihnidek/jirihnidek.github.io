---
title: Kerberos a NFS
layout: post
---

Kerberos a NFS

Nejprve je potřeba jako bežný uživatel zkusit vytvořit Kerberos lístek pomocí:

    kinit jiri_hnidek@EINFRA

Zde budete požádáni o heslo, které používáte k ostatním službám datového úložiště CESNETu (ownclod, filesender, atd.). Pokud heslo nemáte, tak je potřeba se zaregistrovat na: ...

Po úspěšném zadání hesla byste měli být schopni vypsat lístek pomocí:

    klist -l

Výstupem by mělo být něco jako:

    [jiri.hnidek@perseus ~]$ klist -l
    Principal name                 Cache name
    --------------                 ----------
    jmeno_projmeni@EINFRA             FILE:/tmp/krb5cc_1000

Abyste nemuseli pravidelně při pokusu o mapování disků na vizualizační server zadávat heslo, tak je vhodné si vytvořit tzv keytab pomocí:

    ssh -o PubkeyAuthentication=no -o GSSAPIAuthentication=no jmeno_prijmeni@nfs.du3.cesnet.cz "remctl kdc1.cesnet.cz accounts nfskeytab" > krb5.keytab

Následně je možné