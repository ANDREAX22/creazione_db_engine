---
title: Parte 13 - Aggiornamento del Nodo Genitore Dopo una Divisione
date: 2017-11-26
---

Per il prossimo passo del nostro epico viaggio di implementazione dell'albero b, gestiremo la riparazione del nodo genitore dopo aver diviso una foglia. Userò il seguente esempio come riferimento:

{% include image.html url="assets/images/updating-internal-node.png" description="Esempio di aggiornamento nodo interno" %}

In questo esempio, aggiungiamo la chiave "3" all'albero. Questo causa la divisione del nodo foglia sinistro. Dopo la divisione ripariamo l'albero facendo quanto segue:

1. Aggiorniamo la prima chiave nel genitore per essere la chiave massima nel figlio sinistro ("3")
2. Aggiungiamo una nuova coppia puntatore figlio / chiave dopo la chiave aggiornata
  - Il nuovo puntatore punta al nuovo nodo figlio
  - La nuova chiave è la chiave massima nel nuovo nodo figlio ("5")

Quindi prima di tutto, sostituiamo il nostro codice stub con due nuove chiamate di funzione: `update_internal_node_key()` per il passo 1 e `internal_node_insert()` per il passo 2


```diff
@@ -670,9 +725,11 @@ void leaf_node_split_and_insert(Cursor* cursor, uint32_t key, Row* value) {
   */
 
   void* old_node = get_page(cursor->table->pager, cursor->page_num);
+  uint32_t old_max = get_node_max_key(old_node);
   uint32_t new_page_num = get_unused_page_num(cursor->table->pager);
   void* new_node = get_page(cursor->table->pager, new_page_num);
   initialize_leaf_node(new_node);
+  *node_parent(new_node) = *node_parent(old_node);
   *leaf_node_next_leaf(new_node) = *leaf_node_next_leaf(old_node);
   *leaf_node_next_leaf(old_node) = new_page_num;
 
@@ -709,8 +766,12 @@ void leaf_node_split_and_insert(Cursor* cursor, uint32_t key, Row* value) {
   if (is_node_root(old_node)) {
     return create_new_root(cursor->table, new_page_num);
   } else {
-    printf("Need to implement updating parent after split\n");
-    exit(EXIT_FAILURE);
+    uint32_t parent_page_num = *node_parent(old_node);
+    uint32_t new_max = get_node_max_key(old_node);
+    void* parent = get_page(cursor->table->pager, parent_page_num);
+
+    update_internal_node_key(parent, old_max, new_max);
+    internal_node_insert(cursor->table, parent_page_num, new_page_num);
+    return;
   }
 }
```

Per ottenere un riferimento al genitore, dobbiamo iniziare a registrare in ogni nodo un puntatore al suo nodo genitore.

```diff
+uint32_t* node_parent(void* node) { return node + PARENT_POINTER_OFFSET; }
```
```diff
@@ -660,6 +675,48 @@ void create_new_root(Table* table, uint32_t right_child_page_num) {
   uint32_t left_child_max_key = get_node_max_key(left_child);
   *internal_node_key(root, 0) = left_child_max_key;
   *internal_node_right_child(root) = right_child_page_num;
+  *node_parent(left_child) = table->root_page_num;
+  *node_parent(right_child) = table->root_page_num;
 }
```

Ora dobbiamo trovare la cella interessata nel nodo genitore. Il figlio non conosce il proprio numero di pagina, quindi non possiamo cercare quello. Ma conosce la propria chiave massima, quindi possiamo cercare il genitore per quella chiave.

```diff
+void update_internal_node_key(void* node, uint32_t old_key, uint32_t new_key) {
+  uint32_t old_child_index = internal_node_find_child(node, old_key);
+  *internal_node_key(node, old_child_index) = new_key;
 }
```

All'interno di `internal_node_find_child()` riutilizzeremo del codice che abbiamo già per trovare una chiave in un nodo interno. Rifattorizziamo `internal_node_find()` per usare il nuovo metodo helper.

```diff
-Cursor* internal_node_find(Table* table, uint32_t page_num, uint32_t key) {
-  void* node = get_page(table->pager, page_num);
+uint32_t internal_node_find_child(void* node, uint32_t key) {
+  /*
+  Restituisce l'indice del figlio che dovrebbe contenere
+  la chiave data.
+  */
+
   uint32_t num_keys = *internal_node_num_keys(node);
 
-  /* Binary search to find index of child to search */
+  /* Ricerca binaria */
   uint32_t min_index = 0;
   uint32_t max_index = num_keys; /* there is one more child than key */
 
@@ -386,7 +394,14 @@ Cursor* internal_node_find(Table* table, uint32_t page_num, uint32_t key) {
     }
   }
 
-  uint32_t child_num = *internal_node_child(node, min_index);
+  return min_index;
+}
+
+Cursor* internal_node_find(Table* table, uint32_t page_num, uint32_t key) {
+  void* node = get_page(table->pager, page_num);
+
+  uint32_t child_index = internal_node_find_child(node, key);
+  uint32_t child_num = *internal_node_child(node, child_index);
   void* child = get_page(table->pager, child_num);
   switch (get_node_type(child)) {
     case NODE_LEAF:
```

Ora arriviamo al cuore di questo articolo, implementando `internal_node_insert()`. Lo spiegherò a pezzi.

```diff
+void internal_node_insert(Table* table, uint32_t parent_page_num,
+                          uint32_t child_page_num) {
+  /*
+  Aggiunge una nuova coppia figlio/chiave al genitore che corrisponde al figlio
+  */
+
+  void* parent = get_page(table->pager, parent_page_num);
+  void* child = get_page(table->pager, child_page_num);
+  uint32_t child_max_key = get_node_max_key(child);
+  uint32_t index = internal_node_find_child(parent, child_max_key);
+
+  uint32_t original_num_keys = *internal_node_num_keys(parent);
+  *internal_node_num_keys(parent) = original_num_keys + 1;
+
+  if (original_num_keys >= INTERNAL_NODE_MAX_CELLS) {
+    printf("Need to implement splitting internal node\n");
+    exit(EXIT_FAILURE);
+  }
```

L'indice dove dovrebbe essere inserita la nuova cella (coppia figlio/chiave) dipende dalla chiave massima nel nuovo figlio. Nell'esempio che abbiamo guardato, `child_max_key` sarebbe 5 e `index` sarebbe 1.

Se non c'è spazio nel nodo interno per un'altra cella, lancia un errore. Lo implementeremo più tardi.

Ora guardiamo il resto della funzione:

```diff
+
+  uint32_t right_child_page_num = *internal_node_right_child(parent);
+  void* right_child = get_page(table->pager, right_child_page_num);
+
+  if (child_max_key > get_node_max_key(right_child)) {
+    /* Sostituisce il figlio destro */
+    *internal_node_child(parent, original_num_keys) = right_child_page_num;
+    *internal_node_key(parent, original_num_keys) =
+        get_node_max_key(right_child);
+    *internal_node_right_child(parent) = child_page_num;
+  } else {
+    /* Fa spazio per la nuova cella */
+    for (uint32_t i = original_num_keys; i > index; i--) {
+      void* destination = internal_node_cell(parent, i);
+      void* source = internal_node_cell(parent, i - 1);
+      memcpy(destination, source, INTERNAL_NODE_CELL_SIZE);
+    }
+    *internal_node_child(parent, index) = child_page_num;
+    *internal_node_key(parent, index) = child_max_key;
+  }
+}
```

Poiché memorizziamo il puntatore figlio più a destra separatamente dal resto delle coppie figlio/chiave, dobbiamo gestire le cose diversamente se il nuovo figlio diventerà il figlio più a destra.

Nel nostro esempio, entreremmo nel blocco `else`. Prima facciamo spazio per la nuova cella spostando altre celle di uno spazio a destra. (Anche se nel nostro esempio ci sono 0 celle da spostare)

Poi, scriviamo il nuovo puntatore figlio e la chiave nella cella determinata da `index`.

Per ridurre la dimensione dei testcase necessari, sto hardcodando `INTERNAL_NODE_MAX_CELLS` per ora

```diff
@@ -126,6 +126,8 @@ const uint32_t INTERNAL_NODE_KEY_SIZE = sizeof(uint32_t);
 const uint32_t INTERNAL_NODE_CHILD_SIZE = sizeof(uint32_t);
 const uint32_t INTERNAL_NODE_CELL_SIZE =
     INTERNAL_NODE_CHILD_SIZE + INTERNAL_NODE_KEY_SIZE;
+/* Mantieni questo piccolo per i test */
+const uint32_t INTERNAL_NODE_MAX_CELLS = 3;
```

A proposito di test, il nostro test di dataset grande supera il nostro vecchio stub e arriva al nostro nuovo:

```diff
@@ -65,7 +65,7 @@ describe 'database' do
     result = run_script(script)
     expect(result.last(2)).to match_array([
       "db > Executed.",
-      "db > Need to implement updating parent after split",
+      "db > Need to implement splitting internal node",
     ])
```

Molto soddisfacente, lo so.

Aggiungerò un altro test che stampa un albero a quattro nodi. Solo per testare più casi degli id sequenziali, questo test aggiungerà record in un ordine pseudocasuale.

```diff
+  it 'allows printing out the structure of a 4-leaf-node btree' do
+    script = [
+      "insert 18 user18 person18@example.com",
+      "insert 7 user7 person7@example.com",
+      "insert 10 user10 person10@example.com",
+      "insert 29 user29 person29@example.com",
+      "insert 23 user23 person23@example.com",
+      "insert 4 user4 person4@example.com",
+      "insert 14 user14 person14@example.com",
+      "insert 30 user30 person30@example.com",
+      "insert 15 user15 person15@example.com",
+      "insert 26 user26 person26@example.com",
+      "insert 22 user22 person22@example.com",
+      "insert 19 user19 person19@example.com",
+      "insert 2 user2 person2@example.com",
+      "insert 1 user1 person1@example.com",
+      "insert 21 user21 person21@example.com",
+      "insert 11 user11 person11@example.com",
+      "insert 6 user6 person6@example.com",
+      "insert 20 user20 person20@example.com",
+      "insert 5 user5 person5@example.com",
+      "insert 8 user8 person8@example.com",
+      "insert 9 user9 person9@example.com",
+      "insert 3 user3 person3@example.com",
+      "insert 12 user12 person12@example.com",
+      "insert 27 user27 person27@example.com",
+      "insert 17 user17 person17@example.com",
+      "insert 16 user16 person16@example.com",
+      "insert 13 user13 person13@example.com",
+      "insert 24 user24 person24@example.com",
+      "insert 25 user25 person25@example.com",
+      "insert 28 user28 person28@example.com",
+      ".btree",
+      ".exit",
+    ]
+    result = run_script(script)
```

Così com'è, produrrà questo:

```
- internal (size 3)
  - leaf (size 7)
    - 1
    - 2
    - 3
    - 4
    - 5
    - 6
    - 7
  - key 1
  - leaf (size 8)
    - 8
    - 9
    - 10
    - 11
    - 12
    - 13
    - 14
    - 15
  - key 15
  - leaf (size 7)
    - 16
    - 17
    - 18
    - 19
    - 20
    - 21
    - 22
  - key 22
  - leaf (size 8)
    - 23
    - 24
    - 25
    - 26
    - 27
    - 28
    - 29
    - 30
db >
```

Guarda attentamente e noterai un bug:
```
    - 5
    - 6
    - 7
  - key 1
```

La chiave lì dovrebbe essere 7, non 1!

Dopo un po' di debug, ho scoperto che questo era dovuto a una cattiva aritmetica dei puntatori.

```diff
 uint32_t* internal_node_key(void* node, uint32_t key_num) {
-  return internal_node_cell(node, key_num) + INTERNAL_NODE_CHILD_SIZE;
+  return (void*)internal_node_cell(node, key_num) + INTERNAL_NODE_CHILD_SIZE;
 }
```

`INTERNAL_NODE_CHILD_SIZE` è 4. La mia intenzione qui era di aggiungere 4 byte al risultato di `internal_node_cell()`, ma dato che `internal_node_cell()` restituisce un `uint32_t*`, questo stava effettivamente aggiungendo `4 * sizeof(uint32_t)` byte. L'ho risolto facendo il cast a un `void*` prima di fare l'aritmetica.

NOTA! [L'aritmetica dei puntatori sui puntatori void non fa parte dello standard C e potrebbe non funzionare con il tuo compilatore](https://stackoverflow.com/questions/3523145/pointer-arithmetic-for-void-pointer-in-c/46238658#46238658). Potrei fare un articolo in futuro sulla portabilità, ma per ora lascio la mia aritmetica dei puntatori void.

Perfetto. Un altro passo verso un'implementazione btree completamente operativa. Il prossimo passo dovrebbe essere la divisione dei nodi interni. Fino ad allora!
