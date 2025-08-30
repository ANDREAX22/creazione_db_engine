---
title: Parte 10 - Divisione di un Nodo Foglia
date: 2017-10-09
---

Il nostro B-Tree non sembra molto un albero con solo un nodo. Per risolvere questo, abbiamo bisogno di codice per dividere un nodo foglia in due. E dopo quello, dobbiamo creare un nodo interno per servire come genitore per i due nodi foglia.

Fondamentalmente il nostro obiettivo per questo articolo è passare da questo:

{% include image.html url="assets/images/btree2.png" description="btree a un nodo" %}

a questo:

{% include image.html url="assets/images/btree3.png" description="btree a due livelli" %}

Prima di tutto, rimuoviamo la gestione degli errori per un nodo foglia pieno:

```diff
 void leaf_node_insert(Cursor* cursor, uint32_t key, Row* value) {
   void* node = get_page(cursor->table->pager, cursor->page_num);
 
   uint32_t num_cells = *leaf_node_num_cells(node);
   if (num_cells >= LEAF_NODE_MAX_CELLS) {
     // Nodo pieno
-    printf("Need to implement splitting a leaf node.\n");
-    exit(EXIT_FAILURE);
+    leaf_node_split_and_insert(cursor, key, value);
+    return;
   }
```

```diff
ExecuteResult execute_insert(Statement* statement, Table* table) {
   void* node = get_page(table->pager, table->root_page_num);
   uint32_t num_cells = (*leaf_node_num_cells(node));
-  if (num_cells >= LEAF_NODE_MAX_CELLS) {
-    return EXECUTE_TABLE_FULL;
-  }
 
   Row* row_to_insert = &(statement->row_to_insert);
   uint32_t key_to_insert = row_to_insert->id;
```

## Algoritmo di Divisione

La parte facile è finita. Ecco una descrizione di quello che dobbiamo fare da [SQLite Database System: Design and Implementation](https://play.google.com/store/books/details/Sibsankar_Haldar_SQLite_Database_System_Design_and?id=9Z6IQQnX1JEC&hl=en)

> Se non c'è spazio nel nodo foglia, divideremmo le entry esistenti che risiedono lì e quella nuova (che viene inserita) in due metà uguali: metà inferiore e superiore. (Le chiavi nella metà superiore sono strettamente maggiori di quelle nella metà inferiore.) Allochiamo un nuovo nodo foglia, e spostiamo la metà superiore nel nuovo nodo.

Otteniamo un handle al nodo vecchio e creiamo il nuovo nodo:

```diff
+void leaf_node_split_and_insert(Cursor* cursor, uint32_t key, Row* value) {
+  /*
+  Crea un nuovo nodo e sposta metà delle celle.
+  Inserisce il nuovo valore in uno dei due nodi.
+  Aggiorna il genitore o crea un nuovo genitore.
+  */
+
+  void* old_node = get_page(cursor->table->pager, cursor->page_num);
+  uint32_t new_page_num = get_unused_page_num(cursor->table->pager);
+  void* new_node = get_page(cursor->table->pager, new_page_num);
+  initialize_leaf_node(new_node);
```

Poi, copia ogni cella nella sua nuova posizione:

```diff
+  /*
+  Tutte le chiavi esistenti più la nuova chiave dovrebbero essere divise
+  uniformemente tra il nodo vecchio (sinistro) e nuovo (destro).
+  Partendo da destra, sposta ogni chiave nella posizione corretta.
+  */
+  for (int32_t i = LEAF_NODE_MAX_CELLS; i >= 0; i--) {
+    void* destination_node;
+    if (i >= LEAF_NODE_LEFT_SPLIT_COUNT) {
+      destination_node = new_node;
+    } else {
+      destination_node = old_node;
+    }
+    uint32_t index_within_node = i % LEAF_NODE_LEFT_SPLIT_COUNT;
+    void* destination = leaf_node_cell(destination_node, index_within_node);
+
+    if (i == cursor->cell_num) {
+      serialize_row(value, destination);
+    } else if (i > cursor->cell_num) {
+      memcpy(destination, leaf_node_cell(old_node, i - 1), LEAF_NODE_CELL_SIZE);
+    } else {
+      memcpy(destination, leaf_node_cell(old_node, i), LEAF_NODE_CELL_SIZE);
+    }
+  }
```

Aggiorna i conteggi delle celle nell'header di ogni nodo:

```diff
+  /* Aggiorna il conteggio delle celle su entrambi i nodi foglia */
+  *(leaf_node_num_cells(old_node)) = LEAF_NODE_LEFT_SPLIT_COUNT;
+  *(leaf_node_num_cells(new_node)) = LEAF_NODE_RIGHT_SPLIT_COUNT;
```

Poi dobbiamo aggiornare il genitore dei nodi. Se il nodo originale era la radice, non avesse genitore. In quel caso, creiamo un nuovo nodo radice per agire come genitore. Stubberò l'altra branca per ora:

```diff
+  if (is_node_root(old_node)) {
+    return create_new_root(cursor->table, new_page_num);
+  } else {
+    printf("Need to implement updating parent after split\n");
+    exit(EXIT_FAILURE);
+  }
}
```

## Allocazione di Nuove Pagine

Torniamo indietro e definiamo alcune nuove funzioni e costanti. Quando abbiamo creato un nuovo nodo foglia, l'abbiamo messo in una pagina decisa da `get_unused_page_num()`:

```diff
+/*
+Finché non iniziamo a riciclare le pagine libere, le nuove pagine
+sempre andranno alla fine del file del database
+*/
+uint32_t get_unused_page_num(Pager* pager) { return pager->num_pages; }
```

Per ora, assumiamo che in un database con N pagine, i numeri di pagina 0 a N-1 siano allocati. Quindi possiamo sempre allocare il numero di pagina N per le nuove pagine. Eventualmente dopo aver implementato la cancellazione, alcune pagine potrebbero diventare vuote e i loro numeri di pagina non utilizzati. Per essere più efficiente, potremmo riallocare quelle pagine libere.

## Dimensioni del Nodo Foglia

Per mantenere l'albero bilanciato, distribuiremo uniformemente le celle tra i due nuovi nodi. Se un nodo foglia può contenere N celle, allora durante una divisione dobbiamo distribuire N+1 celle tra due nodi (N celle originali più una nuova). Sceglierò arbitrariamente il nodo sinistro per ottenere un'altra cella se N+1 è dispari.

```diff
+const uint32_t LEAF_NODE_RIGHT_SPLIT_COUNT = (LEAF_NODE_MAX_CELLS + 1) / 2;
+const uint32_t LEAF_NODE_LEFT_SPLIT_COUNT =
+    (LEAF_NODE_MAX_CELLS + 1) - LEAF_NODE_RIGHT_SPLIT_COUNT;
```

## Creazione di una Nuova Radice

Ecco come [SQLite Database System](https://play.google.com/store/books/details/Sibsankar_Haldar_SQLite_Database_System_Design_and?id=9Z6IQQnX1JEC&hl=en) spiega il processo di creazione di un nuovo nodo radice:

> Sia N il nodo radice. Prima di tutto, allochiamo due nodi, diciamo L e R. Spostiamo la metà inferiore di N in L e la metà superiore in R. Ora N è vuoto. Aggiungiamo 〈L, K,R〉 in N, dove K è la chiave massima in L. La pagina N rimane la radice. Nota che la profondità dell'albero è aumentata di uno, ma il nuovo albero rimane bilanciato senza violare alcuna proprietà di B+-tree.

A questo punto, abbiamo già allocato il nodo figlio destro e abbiamo spostato la metà superiore in esso. La nostra funzione prende il nodo figlio destro come input e alloca una nuova pagina per memorizzare il nodo figlio sinistro.

```diff
+void create_new_root(Table* table, uint32_t right_child_page_num) {
+  /*
+  Gestisce la divisione della radice.
+  L'antico root copiato nella nuova pagina, diventa il figlio sinistro.
+  L'indirizzo del figlio destro passato come input.
+  Re-inizializza la pagina root per contenere il nuovo nodo radice.
+  Il nuovo nodo radice punta a due figli.
+  */
+
+  void* root = get_page(table->pager, table->root_page_num);
+  void* right_child = get_page(table->pager, right_child_page_num);
+  uint32_t left_child_page_num = get_unused_page_num(table->pager);
+  void* left_child = get_page(table->pager, left_child_page_num);
```

L'antico root viene copiato nel figlio sinistro in modo da poterlo riutilizzare:

```diff
+  /* Il figlio sinistro ha i dati copiati dall'antico root */
+  memcpy(left_child, root, PAGE_SIZE);
+  set_node_root(left_child, false);
```

Infine, inizializziamo la pagina root come un nuovo nodo interno con due figli.

```diff
+  /* Il nodo radice è un nuovo nodo interno con una chiave e due figli */
+  initialize_internal_node(root);
+  set_node_root(root, true);
+  *internal_node_num_keys(root) = 1;
+  *internal_node_child(root, 0) = left_child_page_num;
+  uint32_t left_child_max_key = get_node_max_key(left_child);
+  *internal_node_key(root, 0) = left_child_max_key;
+  *internal_node_right_child(root) = right_child_page_num;
+}
```

## Formato del Nodo Interno

Ora che finalmente stiamo creando un nodo interno, dobbiamo definire il suo layout. Inizia con l'header comune, poi il numero di chiavi che contiene, poi il numero di pagina del suo figlio più a destra. I nodi interni hanno sempre un puntatore figlio in più rispetto al numero di chiavi. Questo puntatore extra è memorizzato nell'header.

```diff
+/*
+ * Header del Nodo Interno Layout
+ */
+const uint32_t INTERNAL_NODE_NUM_KEYS_SIZE = sizeof(uint32_t);
+const uint32_t INTERNAL_NODE_NUM_KEYS_OFFSET = COMMON_NODE_HEADER_SIZE;
+const uint32_t INTERNAL_NODE_RIGHT_CHILD_SIZE = sizeof(uint32_t);
+const uint32_t INTERNAL_NODE_RIGHT_CHILD_OFFSET =
+    INTERNAL_NODE_NUM_KEYS_OFFSET + INTERNAL_NODE_NUM_KEYS_SIZE;
+const uint32_t INTERNAL_NODE_HEADER_SIZE = COMMON_NODE_HEADER_SIZE +
+                                           INTERNAL_NODE_NUM_KEYS_SIZE +
+                                           INTERNAL_NODE_RIGHT_CHILD_SIZE;
```

Il corpo è un array di celle dove ogni cella contiene un puntatore figlio e una chiave. Ogni chiave dovrebbe essere la chiave massima contenuta nel figlio a sinistra.

```diff
+/*
+ * Corpo del Nodo Interno Layout
+ */
+const uint32_t INTERNAL_NODE_KEY_SIZE = sizeof(uint32_t);
+const uint32_t INTERNAL_NODE_CHILD_SIZE = sizeof(uint32_t);
+const uint32_t INTERNAL_NODE_CELL_SIZE =
+    INTERNAL_NODE_CHILD_SIZE + INTERNAL_NODE_KEY_SIZE;
```

In base a queste costanti, ecco come apparirà il layout di un nodo interno:

{% include image.html url="assets/images/internal-node-format.png" description="Il nostro formato interno" %}

Notiamo il nostro enorme fattore di branching. Poiché ogni coppia di puntatore figlio / chiave è così piccola, possiamo adattare 510 chiavi e 511 puntatori figlio in ogni nodo interno. Ciò significa che non dovremo mai navigare molte strati dell'albero per trovare una chiave data!

| # livelli interni dell'albero | max # nodi foglia    | Dimensione di tutti i nodi foglia |
|------------------------------|---------------------|------------------------|
| 0                            | 511^0 = 1           | 4 KB                   |
| 1                            | 511^1 = 512         | ~2 MB                   |
| 2                            | 511^2 = 261,121     | ~1 GB                   |
| 3                            | 511^3 = 133,432,831 | ~550 GB                 |

In realtà, non possiamo memorizzare un intero 4 KB di dati per nodo foglia a causa dell'overhead del header, delle chiavi e dello spazio perso. Ma possiamo cercare attraverso qualcosa come 500 GB di dati caricando solo 4 pagine dal disco. Questo è il motivo per cui l'albero B è una struttura di dati utile per i database.

Ecco i metodi per leggere e scrivere su un nodo interno:

```diff
+uint32_t* internal_node_num_keys(void* node) {
+  return node + INTERNAL_NODE_NUM_KEYS_OFFSET;
+}
+
+uint32_t* internal_node_right_child(void* node) {
+  return node + INTERNAL_NODE_RIGHT_CHILD_OFFSET;
+}
+
+uint32_t* internal_node_cell(void* node, uint32_t cell_num) {
+  return node + INTERNAL_NODE_HEADER_SIZE + cell_num * INTERNAL_NODE_CELL_SIZE;
+}
+
+uint32_t* internal_node_child(void* node, uint32_t child_num) {
+  uint32_t num_keys = *internal_node_num_keys(node);
+  if (child_num > num_keys) {
+    printf("Tried to access child_num %d > num_keys %d\n", child_num, num_keys);
+    exit(EXIT_FAILURE);
+  } else if (child_num == num_keys) {
+    return internal_node_right_child(node);
+  } else {
+    return internal_node_cell(node, child_num);
+  }
+}
+
+uint32_t* internal_node_key(void* node, uint32_t key_num) {
+  return internal_node_cell(node, key_num) + INTERNAL_NODE_CHILD_SIZE;
+}
```

Per un nodo interno, la chiave massima è sempre la sua chiave più a destra. Per un nodo foglia, è la chiave all'indice massimo:

```diff
+uint32_t get_node_max_key(void* node) {
+  switch (get_node_type(node)) {
+    case NODE_INTERNAL:
+      return *internal_node_key(node, *internal_node_num_keys(node) - 1);
+    case NODE_LEAF:
+      return *leaf_node_key(node, *leaf_node_num_cells(node) - 1);
+  }
+}
```

## Tenere traccia della Radice

Stiamo finalmente usando il campo `is_root` nell'header comune del nodo. Ricordiamo che lo usiamo per decidere come dividere un nodo foglia:

```c
  if (is_node_root(old_node)) {
    return create_new_root(cursor->table, new_page_num);
  } else {
    printf("Need to implement updating parent after split\n");
    exit(EXIT_FAILURE);
  }
}
```

Ecco i getter e setter:

```diff
+bool is_node_root(void* node) {
+  uint8_t value = *((uint8_t*)(node + IS_ROOT_OFFSET));
+  return (bool)value;
+}
+
+void set_node_root(void* node, bool is_root) {
+  uint8_t value = is_root;
+  *((uint8_t*)(node + IS_ROOT_OFFSET)) = value;
+}
```


L'inizializzazione di entrambi i tipi di nodi dovrebbe impostare `is_root` a false per default:

```diff
 void initialize_leaf_node(void* node) {
   set_node_type(node, NODE_LEAF);
+  set_node_root(node, false);
   *leaf_node_num_cells(node) = 0;
 }

+void initialize_internal_node(void* node) {
+  set_node_type(node, NODE_INTERNAL);
+  set_node_root(node, false);
+  *internal_node_num_keys(node) = 0;
+}
```

Dobbiamo impostare `is_root` a true quando creiamo il primo nodo della tabella:

```diff
     // Nuovo file del database. Inizializza la pagina 0 come nodo foglia.
     void* root_node = get_page(pager, 0);
     initialize_leaf_node(root_node);
+    set_node_root(root_node, true);
   }
 
   return table;
```

## Stampa dell'Albero

Per aiutarci a visualizzare lo stato del database, dovremmo aggiornare il nostro comando `.btree` per stampare un albero multi-livello.

Sostituirò la funzione `print_leaf_node()` corrente

```diff
-void print_leaf_node(void* node) {
-  uint32_t num_cells = *leaf_node_num_cells(node);
-  printf("leaf (size %d)\n", num_cells);
-  for (uint32_t i = 0; i < num_cells; i++) {
-    uint32_t key = *leaf_node_key(node, i);
-    printf("  - %d : %d\n", i, key);
-  }
-}
```

con una nuova funzione ricorsiva che prende qualsiasi nodo, poi lo stampa e i suoi figli. Prende un livello di indentazione come parametro, che aumenta con ogni chiamata ricorsiva. Aggiungo anche una funzione di aiuto per l'indentazione.

```diff
+void indent(uint32_t level) {
+  for (uint32_t i = 0; i < level; i++) {
+    printf("  ");
+  }
+}
+
+void print_tree(Pager* pager, uint32_t page_num, uint32_t indentation_level) {
+  void* node = get_page(pager, page_num);
+  uint32_t num_keys, child;
+
+  switch (get_node_type(node)) {
+    case (NODE_LEAF):
+      num_keys = *leaf_node_num_cells(node);
+      indent(indentation_level);
+      printf("- leaf (size %d)\n", num_keys);
+      for (uint32_t i = 0; i < num_keys; i++) {
+        indent(indentation_level + 1);
+        printf("- %d\n", *leaf_node_key(node, i));
+      }
+      break;
+    case (NODE_INTERNAL):
+      num_keys = *internal_node_num_keys(node);
+      indent(indentation_level);
+      printf("- internal (size %d)\n", num_keys);
+      for (uint32_t i = 0; i < num_keys; i++) {
+        child = *internal_node_child(node, i);
+        print_tree(pager, child, indentation_level + 1);

+        indent(indentation_level + 1);
+        printf("- key %d\n", *internal_node_key(node, i));
+      }
+      child = *internal_node_right_child(node);
+      print_tree(pager, child, indentation_level + 1);
+      break;
+  }
+}
```

E aggiorniamo la chiamata alla funzione di stampa, passando un livello di indentazione di zero.

```diff
   } else if (strcmp(input_buffer->buffer, ".btree") == 0) {
     printf("Tree:\n");
-    print_leaf_node(get_page(table->pager, 0));
+    print_tree(table->pager, 0, 0);
     return META_COMMAND_SUCCESS;
```

Ecco un test case per la nuova funzionalità di stampa!

```diff
  it 'allows printing out the structure of a 3-leaf-node btree' do
    script = (1..14).map do |i|
      "insert #{i} user#{i} person#{i}@example.com"
    end
    script << ".btree"
    script << "insert 15 user15 person15@example.com"
    script << ".exit"
    result = run_script(script)

    expect(result[14...(result.length)]).to match_array([
      "db > Tree:",
      "- internal (size 1)",
      "  - leaf (size 7)",
      "    - 1",
      "    - 2",
      "    - 3",
      "    - 4",
      "    - 5",
      "    - 6",
      "    - 7",
      "  - key 7",
      "  - leaf (size 7)",
      "    - 8",
      "    - 9",
      "    - 10",
      "    - 11",
      "    - 12",
      "    - 13",
      "    - 14",
      "db > Need to implement searching an internal node",
    ])
  end
```

Il nuovo formato è un po' semplificato, quindi dobbiamo aggiornare il test `.btree` esistente:

```diff
       "db > Executed.",
       "db > Executed.",
       "db > Tree:",
-      "leaf (size 3)",
-      "  - 0 : 1",
-      "  - 1 : 2",
-      "  - 2 : 3",
+      "- leaf (size 3)",
+      "  - 1",
+      "  - 2",
+      "  - 3",
       "db > "
     ])
   end
```

Ecco l'output di `.btree` del nuovo test da solo:

```
Tree:
- internal (size 1)
  - leaf (size 7)
    - 1
    - 2
    - 3
    - 4
    - 5
    - 6
    - 7
  - key 7
  - leaf (size 7)
    - 8
    - 9
    - 10
    - 11
    - 12
    - 13
    - 14
```

Al livello meno indentato, vediamo il nodo radice (un nodo interno). Dice `size 1` perché ha una chiave. Indentato di un livello, vediamo un nodo foglia, una chiave e un altro nodo foglia. La chiave nel nodo radice (7) è la chiave massima nel primo nodo foglia. Ogni chiave maggiore di 7 si trova nel secondo nodo foglia.

## Un Problema Grande

Se sei stato attento, potresti aver notato che abbiamo perso qualcosa di grande. Guarda cosa succede se proviamo a inserire un'altra riga:

```
db > insert 15 user15 person15@example.com
Need to implement searching an internal node
```

Whoops! Chi ha scritto quel TODO? :P

La prossima volta continuerò la grande saga dell'albero B, implementando la ricerca su un albero multi-livello.
