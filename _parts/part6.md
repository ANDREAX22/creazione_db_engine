---
title: Parte 6 - L'Astrazione Cursor
date: 2017-09-10
---

Questa dovrebbe essere una parte più breve dell'ultima. Stiamo solo rifattorizzando un po' per rendere più facile iniziare l'implementazione del B-Tree.

Aggiungeremo un oggetto `Cursor` che rappresenta una posizione nella tabella. Cose che potresti voler fare con i cursori:

- Creare un cursor all'inizio della tabella
- Creare un cursor alla fine della tabella
- Accedere alla riga a cui punta il cursor
- Avanzare il cursor alla riga successiva

Questi sono i comportamenti che implementeremo ora. Più tardi, vorremo anche:

- Cancellare la riga puntata da un cursor
- Modificare la riga puntata da un cursor
- Cercare una tabella per un dato ID, e creare un cursor che punta alla riga con quell'ID

Senza ulteriori indugi, ecco il tipo `Cursor`:

```diff
+typedef struct {
+  Table* table;
+  uint32_t row_num;
+  bool end_of_table;  // Indica una posizione oltre l'ultimo elemento
+} Cursor;
```

Data la nostra struttura dati tabella attuale, tutto quello che serve per identificare una posizione in una tabella è il numero di riga.

Un cursor ha anche un riferimento alla tabella di cui fa parte (così le nostre funzioni cursor possono prendere solo il cursor come parametro).

Infine, ha un booleano chiamato `end_of_table`. Questo è così possiamo rappresentare una posizione oltre la fine della tabella (che è da qualche parte dove potremmo voler inserire una riga).

`table_start()` e `table_end()` creano nuovi cursori:

```diff
+Cursor* table_start(Table* table) {
+  Cursor* cursor = malloc(sizeof(Cursor));
+  cursor->table = table;
+  cursor->row_num = 0;
+  cursor->end_of_table = (table->num_rows == 0);
+
+  return cursor;
+}
+
+Cursor* table_end(Table* table) {
+  Cursor* cursor = malloc(sizeof(Cursor));
+  cursor->table = table;
+  cursor->row_num = table->num_rows;
+  cursor->end_of_table = true;
+
+  return cursor;
+}
```

La nostra funzione `row_slot()` diventerà `cursor_value()`, che restituisce un puntatore alla posizione descritta dal cursor:

```diff
-void* row_slot(Table* table, uint32_t row_num) {
+void* cursor_value(Cursor* cursor) {
+  uint32_t row_num = cursor->row_num;
   uint32_t page_num = row_num / ROWS_PER_PAGE;
-  void* page = get_page(table->pager, page_num);
+  void* page = get_page(cursor->table->pager, page_num);
   uint32_t row_offset = row_num % ROWS_PER_PAGE;
   uint32_t byte_offset = row_offset * ROW_SIZE;
   return page + byte_offset;
 }
```

Avanzare il cursor nella nostra struttura tabella attuale è semplice come incrementare il numero di riga. Questo sarà un po' più complicato in un B-tree.

```diff
+void cursor_advance(Cursor* cursor) {
+  cursor->row_num += 1;
+  if (cursor->row_num >= cursor->table->num_rows) {
+    cursor->end_of_table = true;
+  }
+}
```

Infine possiamo cambiare i nostri metodi "macchina virtuale" per usare l'astrazione cursor. Quando inseriamo una riga, apriamo un cursor alla fine della tabella, scriviamo a quella posizione del cursor, poi chiudiamo il cursor.

```diff
   Row* row_to_insert = &(statement->row_to_insert);
+  Cursor* cursor = table_end(table);

-  serialize_row(row_to_insert, row_slot(table, table->num_rows));
+  serialize_row(row_to_insert, cursor_value(cursor));
   table->num_rows += 1;

+  free(cursor);
+
   return EXECUTE_SUCCESS;
 }
 ```

Quando selezioniamo tutte le righe nella tabella, apriamo un cursor all'inizio della tabella, stampiamo la riga, poi avanziamo il cursor alla riga successiva. Ripetiamo finché non abbiamo raggiunto la fine della tabella.

```diff
 ExecuteResult execute_select(Statement* statement, Table* table) {
+  Cursor* cursor = table_start(table);
+
   Row row;
-  for (uint32_t i = 0; i < table->num_rows; i++) {
-    deserialize_row(row_slot(table, i), &row);
+  while (!(cursor->end_of_table)) {
+    deserialize_row(cursor_value(cursor), &row);
     print_row(&row);
+    cursor_advance(cursor);
   }
+
+  free(cursor);
+
   return EXECUTE_SUCCESS;
 }
```