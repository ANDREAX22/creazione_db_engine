---
title: Parte 12 - Scansione di un B-Tree Multi-Livello
date: 2017-11-11
---

Ora supportiamo la costruzione di un btree multi-livello, ma abbiamo rotto le istruzioni `select` nel processo. Ecco un test case che inserisce 15 righe e poi prova a stamparle.

```diff
+  it 'prints all rows in a multi-level tree' do
+    script = []
+    (1..15).each do |i|
+      script << "insert #{i} user#{i} person#{i}@example.com"
+    end
+    script << "select"
+    script << ".exit"
+    result = run_script(script)
+
+    expect(result[15...result.length]).to match_array([
+      "db > (1, user1, person1@example.com)",
+      "(2, user2, person2@example.com)",
+      "(3, user3, person3@example.com)",
+      "(4, user4, person4@example.com)",
+      "(5, user5, person5@example.com)",
+      "(6, user6, person6@example.com)",
+      "(7, user7, person7@example.com)",
+      "(8, user8, person8@example.com)",
+      "(9, user9, person9@example.com)",
+      "(10, user10, person10@example.com)",
+      "(11, user11, person11@example.com)",
+      "(12, user12, person12@example.com)",
+      "(13, user13, person13@example.com)",
+      "(14, user14, person14@example.com)",
+      "(15, user15, person15@example.com)",
+      "Executed.", "db > ",
+    ])
+  end
```

Ma quando eseguiamo quel test case ora, quello che succede realmente è:

```
db > select
(2, user1, person1@example.com)
Executed.
```

Questo è strano. Sta stampando solo una riga, e quella riga sembra corrotta (nota che l'id non corrisponde all'username).

La stranezza è perché `execute_select()` inizia all'inizio della tabella, e la nostra implementazione attuale di `table_start()` restituisce la cella 0 del nodo radice. Ma la radice del nostro albero ora è un nodo interno che non contiene alcuna riga. I dati che sono stati stampati devono essere rimasti da quando il nodo radice era una foglia. `execute_select()` dovrebbe davvero restituire la cella 0 del nodo foglia più a sinistra.

Quindi eliminiamo la vecchia implementazione:

```diff
-Cursor* table_start(Table* table) {
-  Cursor* cursor = malloc(sizeof(Cursor));
-  cursor->table = table;
-  cursor->page_num = table->root_page_num;
-  cursor->cell_num = 0;
-
-  void* root_node = get_page(table->pager, table->root_page_num);
-  uint32_t num_cells = *leaf_node_num_cells(root_node);
-  cursor->end_of_table = (num_cells == 0);
-
-  return cursor;
-}
```

E aggiungiamo una nuova implementazione che cerca la chiave 0 (la chiave minima possibile). Anche se la chiave 0 non esiste nella tabella, questo metodo restituirà la posizione dell'id più basso (l'inizio del nodo foglia più a sinistra).

```diff
+Cursor* table_start(Table* table) {
+  Cursor* cursor =  table_find(table, 0);
+
+  void* node = get_page(table->pager, cursor->page_num);
+  uint32_t num_cells = *leaf_node_num_cells(node);
+  cursor->end_of_table = (num_cells == 0);
+
+  return cursor;
+}
```

Con queste modifiche, stampa ancora solo le righe di un nodo:

```
db > select
(1, user1, person1@example.com)
(2, user2, person2@example.com)
(3, user3, person3@example.com)
(4, user4, person4@example.com)
(5, user5, person5@example.com)
(6, user6, person6@example.com)
(7, user7, person7@example.com)
Executed.
db >
```

Con 15 entry, il nostro btree consiste di un nodo interno e due nodi foglia, che assomiglia a questo:

{% include image.html url="assets/images/btree3.png" description="struttura del nostro btree" %}

Per scansionare l'intera tabella, dobbiamo saltare al secondo nodo foglia dopo aver raggiunto la fine del primo. Per fare quello, salveremo un nuovo campo nell'header del nodo foglia chiamato "next_leaf", che terrà il numero di pagina del nodo fratello della foglia sulla destra. Il nodo foglia più a destra avrà un valore `next_leaf` di 0 per denotare nessun fratello (la pagina 0 è riservata per il nodo radice della tabella comunque).

Aggiorna il formato dell'header del nodo foglia per includere il nuovo campo:

```diff
 const uint32_t LEAF_NODE_NUM_CELLS_SIZE = sizeof(uint32_t);
 const uint32_t LEAF_NODE_NUM_CELLS_OFFSET = COMMON_NODE_HEADER_SIZE;
-const uint32_t LEAF_NODE_HEADER_SIZE =
-    COMMON_NODE_HEADER_SIZE + LEAF_NODE_NUM_CELLS_SIZE;
+const uint32_t LEAF_NODE_NEXT_LEAF_SIZE = sizeof(uint32_t);
+const uint32_t LEAF_NODE_NEXT_LEAF_OFFSET =
+    LEAF_NODE_NUM_CELLS_OFFSET + LEAF_NODE_NUM_CELLS_SIZE;
+const uint32_t LEAF_NODE_HEADER_SIZE = COMMON_NODE_HEADER_SIZE +
+                                       LEAF_NODE_NUM_CELLS_SIZE +
+                                       LEAF_NODE_NEXT_LEAF_SIZE;
 
 ```

Aggiungi un metodo per accedere al nuovo campo:
```diff
+uint32_t* leaf_node_next_leaf(void* node) {
+  return node + LEAF_NODE_NEXT_LEAF_OFFSET;
+}
```

Imposta `next_leaf` a 0 di default quando inizializziamo un nuovo nodo foglia:

```diff
@@ -322,6 +330,7 @@ void initialize_leaf_node(void* node) {
   set_node_type(node, NODE_LEAF);
   set_node_root(node, false);
   *leaf_node_num_cells(node) = 0;
+  *leaf_node_next_leaf(node) = 0;  // 0 rappresenta nessun fratello
 }
```

Ogni volta che dividiamo un nodo foglia, aggiorniamo i puntatori fratelli. Il fratello della vecchia foglia diventa la nuova foglia, e il fratello della nuova foglia diventa quello che era il fratello della vecchia foglia.

```diff
@@ -659,6 +671,8 @@ void leaf_node_split_and_insert(Cursor* cursor, uint32_t key, Row* value) {
   uint32_t new_page_num = get_unused_page_num(cursor->table->pager);
   void* new_node = get_page(cursor->table->pager, new_page_num);
   initialize_leaf_node(new_node);
+  *leaf_node_next_leaf(new_node) = *leaf_node_next_leaf(old_node);
+  *leaf_node_next_leaf(old_node) = new_page_num;
```

Aggiungere un nuovo campo cambia alcune costanti:
```diff
   it 'prints constants' do
     script = [
       ".constants",
@@ -199,9 +228,9 @@ describe 'database' do
       "db > Constants:",
       "ROW_SIZE: 293",
       "COMMON_NODE_HEADER_SIZE: 6",
-      "LEAF_NODE_HEADER_SIZE: 10",
+      "LEAF_NODE_HEADER_SIZE: 14",
       "LEAF_NODE_CELL_SIZE: 297",
-      "LEAF_NODE_SPACE_FOR_CELLS: 4086",
+      "LEAF_NODE_SPACE_FOR_CELLS: 4082",
       "LEAF_NODE_MAX_CELLS: 13",
       "db > ",
     ])
```

Ora ogni volta che vogliamo avanzare il cursor oltre la fine di un nodo foglia, possiamo controllare se il nodo foglia ha un fratello. Se ce l'ha, salta ad esso. Altrimenti, siamo alla fine della tabella.

```diff
@@ -428,7 +432,15 @@ void cursor_advance(Cursor* cursor) {
 
   cursor->cell_num += 1;
   if (cursor->cell_num >= (*leaf_node_num_cells(node))) {
-    cursor->end_of_table = true;
+    /* Avanza al prossimo nodo foglia */
+    uint32_t next_page_num = *leaf_node_next_leaf(node);
+    if (next_page_num == 0) {
+      /* Questa era la foglia più a destra */
+      cursor->end_of_table = true;
+    } else {
+      cursor->page_num = next_page_num;
+      cursor->cell_num = 0;
+    }
   }
 }
```

Dopo quelle modifiche, stampiamo effettivamente 15 righe...
```
db > select
(1, user1, person1@example.com)
(2, user2, person2@example.com)
(3, user3, person3@example.com)
(4, user4, person4@example.com)
(5, user5, person5@example.com)
(6, user6, person6@example.com)
(7, user7, person7@example.com)
(8, user8, person8@example.com)
(9, user9, person9@example.com)
(10, user10, person10@example.com)
(11, user11, person11@example.com)
(12, user12, person12@example.com)
(13, user13, person13@example.com)
(1919251317, 14, on14@example.com)
(15, user15, person15@example.com)
Executed.
db >
```

...ma una di esse sembra corrotta
```
(1919251317, 14, on14@example.com)
```

Dopo un po' di debug, ho scoperto che è dovuto a un bug in come dividiamo i nodi foglia:

```diff
@@ -676,7 +690,9 @@ void leaf_node_split_and_insert(Cursor* cursor, uint32_t key, Row* value) {
     void* destination = leaf_node_cell(destination_node, index_within_node);
 
     if (i == cursor->cell_num) {
-      serialize_row(value, destination);
+      serialize_row(value,
+                    leaf_node_value(destination_node, index_within_node));
+      *leaf_node_key(destination_node, index_within_node) = key;
     } else if (i > cursor->cell_num) {
       memcpy(destination, leaf_node_cell(old_node, i - 1), LEAF_NODE_CELL_SIZE);
     } else {
```

Ricorda che ogni cella in un nodo foglia consiste prima di una chiave poi di un valore:

{% include image.html url="assets/images/leaf-node-format.png" description="Formato originale nodo foglia" %}

Stavamo scrivendo la nuova riga (valore) all'inizio della cella, dove dovrebbe andare la chiave. Questo significa che parte dell'username stava andando nella sezione per l'id (da qui l'id follemente grande).

Dopo aver risolto quel bug, finalmente stampiamo l'intera tabella come previsto:

```
db > select
(1, user1, person1@example.com)
(2, user2, person2@example.com)
(3, user3, person3@example.com)
(4, user4, person4@example.com)
(5, user5, person5@example.com)
(6, user6, person6@example.com)
(7, user7, person7@example.com)
(8, user8, person8@example.com)
(9, user9, person9@example.com)
(10, user10, person10@example.com)
(11, user11, person11@example.com)
(12, user12, person12@example.com)
(13, user13, person13@example.com)
(14, user14, person14@example.com)
(15, user15, person15@example.com)
Executed.
db >
```

Uff! Un bug dopo l'altro, ma stiamo facendo progressi.

Fino alla prossima volta.
