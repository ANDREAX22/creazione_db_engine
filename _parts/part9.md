---
title: Parte 9 - Ricerca Binaria e Chiavi Duplicate
date: 2017-10-01
---

L'ultima volta abbiamo notato che stiamo ancora memorizzando le chiavi in ordine non ordinato. Risolveremo quel problema, più rilevare e rifiutare chiavi duplicate.

Ora, la nostra funzione `execute_insert()` sceglie sempre di inserire alla fine della tabella. Invece, dovremmo cercare nella tabella il posto giusto per inserire, poi inserire lì. Se la chiave esiste già lì, restituire un errore.

```diff
ExecuteResult execute_insert(Statement* statement, Table* table) {
   void* node = get_page(table->pager, table->root_page_num);
-  if ((*leaf_node_num_cells(node) >= LEAF_NODE_MAX_CELLS)) {
+  uint32_t num_cells = (*leaf_node_num_cells(node));
+  if (num_cells >= LEAF_NODE_MAX_CELLS) {
     return EXECUTE_TABLE_FULL;
   }

   Row* row_to_insert = &(statement->row_to_insert);
-  Cursor* cursor = table_end(table);
+  uint32_t key_to_insert = row_to_insert->id;
+  Cursor* cursor = table_find(table, key_to_insert);
+
+  if (cursor->cell_num < num_cells) {
+    uint32_t key_at_index = *leaf_node_key(node, cursor->cell_num);
+    if (key_at_index == key_to_insert) {
+      return EXECUTE_DUPLICATE_KEY;
+    }
+  }

   leaf_node_insert(cursor, row_to_insert->id, row_to_insert);
```

Non abbiamo più bisogno della funzione `table_end()`.

```diff
-Cursor* table_end(Table* table) {
-  Cursor* cursor = malloc(sizeof(Cursor));
-  cursor->table = table;
-  cursor->page_num = table->root_page_num;
-
-  void* root_node = get_page(table->pager, table->root_page_num);
-  uint32_t num_cells = *leaf_node_num_cells(root_node);
-  cursor->cell_num = num_cells;
-  cursor->end_of_table = true;
-
-  return cursor;
-}
```

La sostituiremo con un metodo che cerca nell'albero una data chiave.

```diff
+/*
+Restituisce la posizione della chiave data.
+Se la chiave non è presente, restituisce la posizione
+dove dovrebbe essere inserita
+*/
+Cursor* table_find(Table* table, uint32_t key) {
+  uint32_t root_page_num = table->root_page_num;
+  void* root_node = get_page(table->pager, root_page_num);
+
+  if (get_node_type(root_node) == NODE_LEAF) {
+    return leaf_node_find(table, root_page_num, key);
+  } else {
+    printf("Need to implement searching an internal node\n");
+    exit(EXIT_FAILURE);
+  }
}
```

Sto creando uno stub per il ramo dei nodi interni perché non abbiamo ancora implementato i nodi interni. Possiamo cercare il nodo foglia con la ricerca binaria.

```diff
+Cursor* leaf_node_find(Table* table, uint32_t page_num, uint32_t key) {
+  void* node = get_page(table->pager, page_num);
+  uint32_t num_cells = *leaf_node_num_cells(node);
+
+  Cursor* cursor = malloc(sizeof(Cursor));
+  cursor->table = table;
+  cursor->page_num = page_num;
+
+  // Ricerca binaria
+  uint32_t min_index = 0;
+  uint32_t one_past_max_index = num_cells;
+  while (one_past_max_index != min_index) {
+    uint32_t index = (min_index + one_past_max_index) / 2;
+    uint32_t key_at_index = *leaf_node_key(node, index);
+    if (key == key_at_index) {
+      cursor->cell_num = index;
+      return cursor;
+    }
+    if (key < key_at_index) {
+      one_past_max_index = index;
+    } else {
+      min_index = index + 1;
+    }
+  }
+
+  cursor->cell_num = min_index;
+  return cursor;
+}
```

Questo restituirà
- la posizione della chiave,
- la posizione di un'altra chiave che dovremo spostare se vogliamo inserire la nuova chiave, o
- la posizione oltre l'ultima chiave

Dato che ora stiamo controllando il tipo di nodo, abbiamo bisogno di funzioni per ottenere e impostare quel valore in un nodo.

```diff
+NodeType get_node_type(void* node) {
+  uint8_t value = *((uint8_t*)(node + NODE_TYPE_OFFSET));
+  return (NodeType)value;
+}
+
+void set_node_type(void* node, NodeType type) {
+  uint8_t value = type;
+  *((uint8_t*)(node + NODE_TYPE_OFFSET)) = value;
+}
```

Dobbiamo fare il cast a `uint8_t` prima per assicurarci che sia serializzato come un singolo byte.

Abbiamo anche bisogno di inizializzare il tipo di nodo.

```diff
-void initialize_leaf_node(void* node) { *leaf_node_num_cells(node) = 0; }
+void initialize_leaf_node(void* node) {
+  set_node_type(node, NODE_LEAF);
+  *leaf_node_num_cells(node) = 0;
+}
```

Infine, dobbiamo creare e gestire un nuovo codice di errore.

```diff
-enum ExecuteResult_t { EXECUTE_SUCCESS, EXECUTE_TABLE_FULL };
+enum ExecuteResult_t {
+  EXECUTE_SUCCESS,
+  EXECUTE_DUPLICATE_KEY,
+  EXECUTE_TABLE_FULL
+};
```

```diff
       case (EXECUTE_SUCCESS):
         printf("Executed.\n");
         break;
+      case (EXECUTE_DUPLICATE_KEY):
+        printf("Error: Duplicate key.\n");
+        break;
       case (EXECUTE_TABLE_FULL):
         printf("Error: Table full.\n");
         break;
```

Con queste modifiche, il nostro test può cambiare per controllare l'ordine ordinato:

```diff
       "db > Executed.",
       "db > Tree:",
       "leaf (size 3)",
-      "  - 0 : 3",
-      "  - 1 : 1",
-      "  - 2 : 2",
+      "  - 0 : 1",
+      "  - 1 : 2",
+      "  - 2 : 3",
       "db > "
     ])
   end
```

E possiamo aggiungere un nuovo test per chiavi duplicate:

```diff
+  it 'prints an error message if there is a duplicate id' do
+    script = [
+      "insert 1 user1 person1@example.com",
+      "insert 1 user1 person1@example.com",
+      "select",
+      ".exit",
+    ]
+    result = run_script(script)
+    expect(result).to match_array([
+      "db > Executed.",
+      "db > Error: Duplicate key.",
+      "db > (1, user1, person1@example.com)",
+      "Executed.",
+      "db > ",
+    ])
+  end
```

Questo è tutto! Prossimo: implementare la divisione dei nodi foglia e la creazione di nodi interni.
