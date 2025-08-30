---
title: Parte 8 - Formato Nodo Foglia B-Tree
date: 2017-09-25
---

Stiamo cambiando il formato della nostra tabella da un array non ordinato di righe a un B-Tree. Questo è un cambiamento abbastanza grande che richiederà più articoli per essere implementato. Alla fine di questo articolo, definiremo il layout di un nodo foglia e supporteremo l'inserimento di coppie chiave/valore in un albero a singolo nodo. Ma prima, ricapitoliamo i motivi per passare a una struttura ad albero.

## Formati Tabella Alternativi

Con il formato attuale, ogni pagina memorizza solo righe (nessun metadata) quindi è abbastanza efficiente nello spazio. L'inserimento è anche veloce perché semplicemente aggiungiamo alla fine. Tuttavia, trovare una riga particolare può essere fatto solo scansionando l'intera tabella. E se vogliamo cancellare una riga, dobbiamo riempire il buco spostando ogni riga che viene dopo.

Se memorizzassimo la tabella come un array, ma mantenessimo le righe ordinate per id, potremmo usare la ricerca binaria per trovare un id particolare. Tuttavia, l'inserimento sarebbe lento perché dovremmo spostare molte righe per fare spazio.

Invece, andremo con una struttura ad albero. Ogni nodo nell'albero può contenere un numero variabile di righe, quindi dobbiamo memorizzare alcune informazioni in ogni nodo per tenere traccia di quante righe contiene. Inoltre c'è l'overhead di memorizzazione di tutti i nodi interni che non memorizzano alcuna riga. In cambio di un file di database più grande, otteniamo inserimento, cancellazione e ricerca veloci.

|               | Array non ordinato di righe | Array ordinato di righe | Albero di nodi |
|---------------|------------------------------|--------------------------|----------------|
| Le pagine contengono | solo dati                  | solo dati               | metadata, chiavi primarie e dati |
| Righe per pagina | più                         | più                     | meno           |
| Inserimento   | O(1)                        | O(n)                    | O(log(n))      |
| Cancellazione | O(n)                        | O(n)                    | O(log(n))      |
| Ricerca per id | O(n)                        | O(log(n))               | O(log(n))      |

## Formato Header Nodo

I nodi foglia e i nodi interni hanno layout diversi. Facciamo un enum per tenere traccia del tipo di nodo:

```diff
+typedef enum { NODE_INTERNAL, NODE_LEAF } NodeType;
```

Ogni nodo corrisponderà a una pagina. I nodi interni punteranno ai loro figli memorizzando il numero di pagina che memorizza il figlio. Il btree chiede al pager un particolare numero di pagina e ottiene indietro un puntatore nella cache delle pagine. Le pagine sono memorizzate nel file del database una dopo l'altra in ordine di numero di pagina.

I nodi devono memorizzare alcuni metadata in un header all'inizio della pagina. Ogni nodo memorizzerà che tipo di nodo è, se è o meno il nodo radice, e un puntatore al suo genitore (per permettere di trovare i fratelli di un nodo). Definisco costanti per la dimensione e l'offset di ogni campo dell'header:

```diff
+/*
+ * Layout Header Nodo Comune
+ */
+const uint32_t NODE_TYPE_SIZE = sizeof(uint8_t);
+const uint32_t NODE_TYPE_OFFSET = 0;
+const uint32_t IS_ROOT_SIZE = sizeof(uint8_t);
+const uint32_t IS_ROOT_OFFSET = NODE_TYPE_SIZE;
+const uint32_t PARENT_POINTER_SIZE = sizeof(uint32_t);
+const uint32_t PARENT_POINTER_OFFSET = IS_ROOT_OFFSET + IS_ROOT_SIZE;
+const uint8_t COMMON_NODE_HEADER_SIZE =
+    NODE_TYPE_SIZE + IS_ROOT_SIZE + PARENT_POINTER_SIZE;
```

## Formato Nodo Foglia

Oltre a questi campi header comuni, i nodi foglia devono memorizzare quante "celle" contengono. Una cella è una coppia chiave/valore.

```diff
+/*
+ * Layout Header Nodo Foglia
+ */
+const uint32_t LEAF_NODE_NUM_CELLS_SIZE = sizeof(uint32_t);
+const uint32_t LEAF_NODE_NUM_CELLS_OFFSET = COMMON_NODE_HEADER_SIZE;
+const uint32_t LEAF_NODE_HEADER_SIZE =
+    COMMON_NODE_HEADER_SIZE + LEAF_NODE_NUM_CELLS_SIZE;
```

Il corpo di un nodo foglia è un array di celle. Ogni cella è una chiave seguita da un valore (una riga serializzata).

```diff
+/*
+ * Layout Corpo Nodo Foglia
+ */
+const uint32_t LEAF_NODE_KEY_SIZE = sizeof(uint32_t);
+const uint32_t LEAF_NODE_KEY_OFFSET = 0;
+const uint32_t LEAF_NODE_VALUE_SIZE = ROW_SIZE;
+const uint32_t LEAF_NODE_VALUE_OFFSET =
+    LEAF_NODE_KEY_OFFSET + LEAF_NODE_KEY_SIZE;
+const uint32_t LEAF_NODE_CELL_SIZE = LEAF_NODE_KEY_SIZE + LEAF_NODE_VALUE_SIZE;
+const uint32_t LEAF_NODE_SPACE_FOR_CELLS = PAGE_SIZE - LEAF_NODE_HEADER_SIZE;
+const uint32_t LEAF_NODE_MAX_CELLS =
+    LEAF_NODE_SPACE_FOR_CELLS / LEAF_NODE_CELL_SIZE;
```

Basandoci su queste costanti, ecco come appare attualmente il layout di un nodo foglia:

{% include image.html url="assets/images/leaf-node-format.png" description="Il nostro formato nodo foglia" %}

È un po' inefficiente nello spazio usare un intero byte per valore booleano nell'header, ma questo rende più facile scrivere codice per accedere a quei valori.

Inoltre notate che c'è dello spazio sprecato alla fine. Memorizziamo quante celle possiamo dopo l'header, ma lo spazio rimanente non può contenere un'intera cella. Lo lasciamo vuoto per evitare di dividere celle tra nodi.

## Accesso ai Campi Nodo Foglia

Il codice per accedere a chiavi, valori e metadata coinvolge tutti l'aritmetica dei puntatori usando le costanti che abbiamo appena definito.

```diff
+uint32_t* leaf_node_num_cells(void* node) {
+  return node + LEAF_NODE_NUM_CELLS_OFFSET;
+}
+
+void* leaf_node_cell(void* node, uint32_t cell_num) {
+  return node + LEAF_NODE_HEADER_SIZE + cell_num * LEAF_NODE_CELL_SIZE;
+}
```
