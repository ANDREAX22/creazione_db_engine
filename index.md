---
title: Come Funziona un Database?
---

- In che formato vengono salvati i dati? (in memoria e su disco)
- Quando si spostano dalla memoria al disco?
- Perché può esserci solo una chiave primaria per tabella?
- Come funziona il rollback di una transazione?
- Come sono formattati gli indici?
- Quando e come avviene una scansione completa della tabella?
- In che formato viene salvata una prepared statement?

In breve, come **funziona** un database?

Sto costruendo un clone di [sqlite](https://www.sqlite.org/arch.html) da zero in C per capire, e documenterò il mio processo mentre procedo.

# Indice
{% for part in site.parts %}- [{{part.title}}]({{site.baseurl}}{{part.url}})
{% endfor %}

> "Quello che non posso creare, non lo capisco." -- [Richard Feynman](https://en.m.wikiquote.org/wiki/Richard_Feynman)

{% include image.html url="assets/images/arch2.gif" description="architettura sqlite (https://www.sqlite.org/arch.html)" %}
