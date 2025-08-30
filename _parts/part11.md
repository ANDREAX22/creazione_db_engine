---
title: Parte 11 - Ricerca Ricorsiva del B-Tree
date: 2017-10-22
---

L'ultima volta abbiamo finito con un errore inserendo la nostra 15ª riga:

```
db > insert 15 user15 person15@example.com
Need to implement searching an internal node
```

Prima, sostituiamo il codice stub con una nuova chiamata di funzione.

```diff
   if (get_node_type(root_node) == NODE_LEAF) {
     return leaf_node_find(table, root_page_num, key);
   } else {
-    printf("Need to implement searching an internal node\n");
-    exit(EXIT_FAILURE);
+    return internal_node_find(table, root_page_num, key);
   }
 }
```

Questa funzione eseguirà la ricerca binaria per trovare il figlio che dovrebbe contenere la chiave data. Ricorda che la chiave alla destra di ogni puntatore figlio è la chiave massima contenuta da quel figlio.

{% include image.html url="assets/images/btree6.png" description="btree a tre livelli" %}

Quindi la nostra ricerca binaria confronta la chiave da trovare e la chiave alla destra del puntatore figlio:

```diff
+Cursor* internal_node_find(Table* table, uint32_t page_num, uint32_t key) {
+  void* node = get_page(table->pager, page_num);
+  uint32_t num_keys = *internal_node_num_keys(node);
+
+  /* Ricerca binaria per trovare l'indice del figlio da cercare */
+  uint32_t min_index = 0;
+  uint32_t max_index = num_keys; /* c'è un figlio in più delle chiavi */
+
+  while (min_index != max_index) {
+    uint32_t index = (min_index + max_index) / 2;
+    uint32_t key_to_right = *internal_node_key(node, index);
+    if (key_to_right >= key) {
+      max_index = index;
+    } else {
+      min_index = index + 1;
+    }
+  }
```

Ricorda anche che i figli di un nodo interno possono essere nodi foglia o più nodi interni. Dopo aver trovato il figlio corretto, chiamiamo la funzione di ricerca appropriata su di esso:

```diff
+  uint32_t child_num = *internal_node_child(node, min_index);
+  void* child = get_page(table->pager, child_num);
+  switch (get_node_type(child)) {
+    case NODE_LEAF:
+      return leaf_node_find(table, child_num, key);
+    case NODE_INTERNAL:
+      return internal_node_find(table, child_num, key);
+  }
+}
```

# Test

Ora inserire una chiave in un btree multi-nodo non risulta più in un errore. E possiamo aggiornare il nostro test:

```diff
       "    - 12",
       "    - 13",
       "    - 14",
-      "db > Need to implement searching an internal node",
+      "db > Executed.",
+      "db > ",
     ])
   end
```

Penso anche che sia tempo di rivisitare un altro test. Quello che prova a inserire 1400 righe. Ancora dà errore, ma il messaggio di errore è nuovo. Ora, i nostri test non gestiscono molto bene quando il programma si blocca. Se succede, usiamo semplicemente l'output che abbiamo ottenuto finora:

```diff
     raw_output = nil
     IO.popen("./db test.db", "r+") do |pipe|
       commands.each do |command|
-        pipe.puts command
+        begin
+          pipe.puts command
+        rescue Errno::EPIPE
+          break
+        end
       end

       pipe.close_write
```

E questo rivela che il nostro test di 1400 righe produce questo errore:

```diff
     end
     script << ".exit"
     result = run_script(script)
-    expect(result[-2]).to eq('db > Error: Table full.')
+    expect(result.last(2)).to match_array([
+      "db > Executed.",
+      "db > Need to implement updating parent after split",
+    ])
   end
```

Sembra che quello sia il prossimo nella nostra lista di cose da fare!

