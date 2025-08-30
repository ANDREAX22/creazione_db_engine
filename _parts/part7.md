---
title: Parte 7 - Introduzione al B-Tree
date: 2017-09-23
---

Il B-Tree è la struttura dati che SQLite usa per rappresentare sia tabelle che indici, quindi è un'idea abbastanza centrale. Questo articolo introdurrà solo la struttura dati, quindi non avrà codice.

Perché un albero è una buona struttura dati per un database?

- Cercare un valore particolare è veloce (tempo logaritmico)
- Inserire/cancellare un valore che hai già trovato è veloce (tempo costante per ribilanciare)
- Attraversare un intervallo di valori è veloce (a differenza di una hash map)

Un B-Tree è diverso da un albero binario (la "B" probabilmente sta per il nome dell'inventore, ma potrebbe anche stare per "balanced"). Ecco un esempio di B-Tree:

{% include image.html url="assets/images/B-tree.png" description="esempio B-Tree (https://en.wikipedia.org/wiki/File:B-tree.svg)" %}

A differenza di un albero binario, ogni nodo in un B-Tree può avere più di 2 figli. Ogni nodo può avere fino a m figli, dove m è chiamato "ordine" dell'albero. Per mantenere l'albero per lo più bilanciato, diciamo anche che i nodi devono avere almeno m/2 figli (arrotondato per eccesso).

Eccezioni:
- I nodi foglia hanno 0 figli
- Il nodo radice può avere meno di m figli ma deve averne almeno 2
- Se il nodo radice è un nodo foglia (l'unico nodo), ha ancora 0 figli

L'immagine sopra è un B-Tree, che SQLite usa per memorizzare indici. Per memorizzare tabelle, SQLite usa una variazione chiamata B+ tree.

|                               | B-tree         | B+ tree             |
|-------------------------------|----------------|---------------------|
| Pronunciato                   | "Bee Tree"     | "Bee Plus Tree"     |
| Usato per memorizzare         | Indici         | Tabelle             |
| I nodi interni memorizzano chiavi | Sì            | Sì                 |
| I nodi interni memorizzano valori | Sì            | No                 |
| Numero di figli per nodo      | Meno           | Più                 |
| Nodi interni vs nodi foglia   | Stessa struttura | Struttura diversa |

Finché non arriviamo a implementare gli indici, parlerò solo di B+ tree, ma mi riferirò ad esso semplicemente come B-tree o btree.

I nodi con figli sono chiamati nodi "interni". I nodi interni e i nodi foglia sono strutturati diversamente:

| Per un albero di ordine m... | Nodo Interno                 | Nodo Foglia         |
|------------------------------|------------------------------|---------------------|
| Memorizza                    | chiavi e puntatori ai figli | chiavi e valori     |
| Numero di chiavi             | fino a m-1                   | quante ne entrano   |
| Numero di puntatori          | numero di chiavi + 1         | nessuno             |
| Numero di valori             | nessuno                      | numero di chiavi    |
| Scopo delle chiavi           | usate per il routing         | accoppiate con valore |
| Memorizza valori?            | No                           | Sì                 |

Lavoriamo attraverso un esempio per vedere come un B-tree cresce mentre inserisci elementi in esso. Per mantenere le cose semplici, l'albero sarà di ordine 3. Questo significa:

- fino a 3 figli per nodo interno
- fino a 2 chiavi per nodo interno
- almeno 2 figli per nodo interno
- almeno 1 chiave per nodo interno

Un B-tree vuoto ha un singolo nodo: il nodo radice. Il nodo radice inizia come nodo foglia con zero coppie chiave/valore:

{% include image.html url="assets/images/btree1.png" description="btree vuoto" %}

Se inseriamo un paio di coppie chiave/valore, sono memorizzate nel nodo foglia in ordine ordinato.

{% include image.html url="assets/images/btree2.png" description="btree a un nodo" %}

Diciamo che la capacità di un nodo foglia è di due coppie chiave/valore. Quando ne inseriamo un'altra, dobbiamo dividere il nodo foglia e mettere metà delle coppie in ogni nodo. Entrambi i nodi diventano figli di un nuovo nodo interno che ora sarà il nodo radice.

{% include image.html url="assets/images/btree3.png" description="btree a due livelli" %}

Il nodo interno ha 1 chiave e 2 puntatori ai nodi figli. Se vogliamo cercare una chiave che è minore o uguale a 5, guardiamo nel figlio sinistro. Se vogliamo cercare una chiave maggiore di 5, guardiamo nel figlio destro.

Ora inseriamo la chiave "2". Prima cerchiamo in quale nodo foglia sarebbe se fosse presente, e arriviamo al nodo foglia sinistro. Il nodo è pieno, quindi dividiamo il nodo foglia e creiamo una nuova entry nel nodo genitore.

{% include image.html url="assets/images/btree4.png" description="btree a quattro nodi" %}

Continuiamo ad aggiungere chiavi. 18 e 21. Arriviamo al punto dove dobbiamo dividere di nuovo, ma non c'è spazio nel nodo genitore per un'altra coppia chiave/puntatore.

{% include image.html url="assets/images/btree5.png" description="nessuno spazio nel nodo interno" %}

La soluzione è dividere il nodo radice in due nodi interni, poi creare un nuovo nodo radice per essere il loro genitore.

{% include image.html url="assets/images/btree6.png" description="btree a tre livelli" %}

La profondità dell'albero aumenta solo quando dividiamo il nodo radice. Ogni nodo foglia ha la stessa profondità e un numero simile di coppie chiave/valore, quindi l'albero rimane bilanciato e veloce da cercare.

Rimanderò la discussione sulla cancellazione di chiavi dall'albero finché non avremo implementato l'inserimento.

Quando implementeremo questa struttura dati, ogni nodo corrisponderà a una pagina. Il nodo radice esisterà nella pagina 0. I puntatori ai figli saranno semplicemente il numero di pagina che contiene il nodo figlio.

La prossima volta, iniziamo a implementare il btree!
