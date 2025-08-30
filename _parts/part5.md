---
title: Parte 5 - Persistenza su Disco
date: 2017-09-08
---

> "Niente al mondo può prendere il posto della persistenza." -- [Calvin Coolidge](https://en.wikiquote.org/wiki/Calvin_Coolidge)

Il nostro database ti permette di inserire record e leggerli di nuovo, ma solo finché mantieni il programma in esecuzione. Se uccidi il programma e lo riavvii, tutti i tuoi record sono andati. Ecco una specifica per il comportamento che vogliamo:

```ruby
it 'keeps data after closing connection' do
  result1 = run_script([
    "insert 1 user1 person1@example.com",
    ".exit",
  ])
  expect(result1).to match_array([
    "db > Executed.",
    "db > ",
  ])
  result2 = run_script([
    "select",
    ".exit",
  ])
  expect(result2).to match_array([
    "db > (1, user1, person1@example.com)",
    "Executed.",
    "db > ",
  ])
end
```

Come sqlite, persisteremo i record salvando l'intero database in un file.

Ci siamo già preparati per farlo serializzando le righe in blocchi di memoria della dimensione di una pagina. Per aggiungere la persistenza, possiamo semplicemente scrivere quei blocchi di memoria in un file, e leggerli di nuovo in memoria la prossima volta che il programma si avvia.

Per rendere questo più facile, creeremo un'astrazione chiamata pager. Chiediamo al pager la pagina numero `x`, e il pager ci restituisce un blocco di memoria. Prima guarda nella sua cache. In caso di cache miss, copia i dati dal disco in memoria (leggendo il file del database).

{% include image.html url="assets/images/arch-part5.gif" description="Come il nostro programma si allinea con l'architettura SQLite" %}

Il Pager accede alla cache delle pagine e al file. L'oggetto Table fa richieste di pagine attraverso il pager:

```diff
+typedef struct {
+  int file_descriptor;
+  uint32_t file_length;
+  void* pages[TABLE_MAX_PAGES];
+} Pager;
+
 typedef struct {
-  void* pages[TABLE_MAX_PAGES];
+  Pager* pager;
   uint32_t num_rows;
 } Table;
```

Sto rinominando `new_table()` in `db_open()` perché ora ha l'effetto di aprire una connessione al database. Per aprire una connessione, intendo:

- aprire il file del database
- inizializzare una struttura dati pager
- inizializzare una struttura dati tabella

```diff
-Table* new_table() {
+Table* db_open(const char* filename) {
+  Pager* pager = pager_open(filename);
+  uint32_t num_rows = pager->file_length / ROW_SIZE;
+
   Table* table = malloc(sizeof(Table));
-  table->num_rows = 0;
+  table->pager = pager;
+  table->num_rows = num_rows;

   return table;
 }
```

`db_open()` a sua volta chiama `pager_open()`, che apre il file del database e tiene traccia della sua dimensione. Inizializza anche la cache delle pagine a tutti `NULL`.

```diff
+Pager* pager_open(const char* filename) {
+  int fd = open(filename,
+                O_RDWR |      // Modalità Read/Write
+                    O_CREAT,  // Crea file se non esiste
+                S_IWUSR |     // Permesso di scrittura utente
+                    S_IRUSR   // Permesso di lettura utente
+                );

  if (fd == -1) {
    printf("Unable to open file\n");
    exit(EXIT_FAILURE);
  }

  off_t file_length = lseek(fd, 0, SEEK_END);

  Pager* pager = malloc(sizeof(Pager));
  pager->file_descriptor = fd;
  pager->file_length = file_length;

  for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++) {
    pager->pages[i] = NULL;
  }

  return pager;
}
```

Seguendo la nostra nuova astrazione, spostiamo la logica per recuperare una pagina nel suo metodo:

```diff
 void* row_slot(Table* table, uint32_t row_num) {
   uint32_t page_num = row_num / ROWS_PER_PAGE;
-  void* page = table->pages[page_num];
-  if (page == NULL) {
-    // Alloca memoria solo quando proviamo ad accedere alla pagina
-    page = table->pages[page_num] = malloc(PAGE_SIZE);
-  }
+  void* page = get_page(table->pager, page_num);
   uint32_t row_offset = row_num % ROWS_PER_PAGE;
   uint32_t byte_offset = row_offset * ROW_SIZE;
   return page + byte_offset;
 }
```

Il metodo `get_page()` ha la logica per gestire un cache miss. Assumiamo che le pagine siano salvate una dopo l'altra nel file del database: Pagina 0 all'offset 0, pagina 1 all'offset 4096, pagina 2 all'offset 8192, ecc. Se la pagina richiesta si trova fuori dai limiti del file, sappiamo che dovrebbe essere vuota, quindi allochiamo semplicemente della memoria e la restituiamo. La pagina sarà aggiunta al file quando svuoteremo la cache su disco più tardi.

```diff
+void* get_page(Pager* pager, uint32_t page_num) {
+  if (page_num > TABLE_MAX_PAGES) {
+    printf("Tried to fetch page number out of bounds. %d > %d\n", page_num,
+           TABLE_MAX_PAGES);
+    exit(EXIT_FAILURE);
+  }

  if (pager->pages[page_num] == NULL) {
    // Cache miss. Alloca memoria e carica dal file.
    void* page = malloc(PAGE_SIZE);
    uint32_t num_pages = pager->file_length / PAGE_SIZE;

    // Potremmo salvare una pagina parziale alla fine del file
    if (pager->file_length % PAGE_SIZE) {
      num_pages += 1;
    }

    if (page_num <= num_pages) {
      lseek(pager->file_descriptor, page_num * PAGE_SIZE, SEEK_SET);
      ssize_t bytes_read = read(pager->file_descriptor, page, PAGE_SIZE);
      if (bytes_read == -1) {
        printf("Error reading file: %d\n", errno);
        exit(EXIT_FAILURE);
      }
    }

    pager->pages[page_num] = page;
  }

  return pager->pages[page_num];
}
```

Per ora, aspetteremo a svuotare la cache su disco finché l'utente non chiude la connessione al database. Quando l'utente esce, chiameremo un nuovo metodo chiamato `db_close()`, che

- svuota la cache delle pagine su disco
- chiude il file del database
- libera la memoria per le strutture dati Pager e Table

```diff
+void db_close(Table* table) {
+  Pager* pager = table->pager;
+  uint32_t num_full_pages = table->num_rows / ROWS_PER_PAGE;

  for (uint32_t i = 0; i < num_full_pages; i++) {
    if (pager->pages[i] == NULL) {
      continue;
    }
    pager_flush(pager, i, PAGE_SIZE);
    free(pager->pages[i]);
    pager->pages[i] = NULL;
  }

  // Potrebbe esserci una pagina parziale da scrivere alla fine del file
  // Questo non dovrebbe essere necessario dopo che passeremo a un B-tree
  uint32_t num_additional_rows = table->num_rows % ROWS_PER_PAGE;
  if (num_additional_rows > 0) {
    uint32_t page_num = num_full_pages;
    if (pager->pages[page_num] != NULL) {
      pager_flush(pager, page_num, num_additional_rows * ROW_SIZE);
      free(pager->pages[page_num]);
      pager->pages[page_num] = NULL;
    }
  }

  int result = close(pager->file_descriptor);
  if (result == -1) {
    printf("Error closing db file.\n");
    exit(EXIT_FAILURE);
  }
  for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++) {
    void* page = pager->pages[i];
    if (page) {
      free(page);
      pager->pages[i] = NULL;
    }
  }
  free(pager);
  free(table);
}

-MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
+MetaCommandResult do_meta_command(InputBuffer* input_buffer, Table* table) {
   if (strcmp(input_buffer->buffer, ".exit") == 0) {
     close_input_buffer(input_buffer);
     db_close(table);
     exit(EXIT_SUCCESS);
   } else {
     return META_COMMAND_UNRECOGNIZED_COMMAND;
```

Nel nostro design attuale, la lunghezza del file codifica quante righe ci sono nel database, quindi dobbiamo scrivere una pagina parziale alla fine del file. Ecco perché `pager_flush()` prende sia un numero di pagina che una dimensione. Non è il design migliore, ma scomparirà abbastanza velocemente quando inizieremo a implementare il B-tree.

```diff
+void pager_flush(Pager* pager, uint32_t page_num, uint32_t size) {
+  if (pager->pages[page_num] == NULL) {
+    printf("Tried to flush null page\n");
+    exit(EXIT_FAILURE);
+  }

  off_t offset = lseek(pager->file_descriptor, page_num * PAGE_SIZE, SEEK_SET);

  if (offset == -1) {
    printf("Error seeking: %d\n", errno);
    exit(EXIT_FAILURE);
  }

  ssize_t bytes_written =
      write(pager->file_descriptor, pager->pages[page_num], size);

  if (bytes_written == -1) {
    printf("Error writing: %d\n", errno);
    exit(EXIT_FAILURE);
  }
}
```

Infine, dobbiamo accettare il nome del file come argomento della riga di comando. Non dimenticare di aggiungere anche l'argomento extra a `do_meta_command`:

```diff
 int main(int argc, char* argv[]) {
-  Table* table = new_table();
+  if (argc < 2) {
+    printf("Must supply a database filename.\n");
+    exit(EXIT_FAILURE);
+  }

  char* filename = argv[1];
  Table* table = db_open(filename);

  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
```
Con queste modifiche, siamo in grado di chiudere e poi riaprire il database, e i nostri record sono ancora lì!

```
~ ./db mydb.db
db > insert 1 cstack foo@bar.com
Executed.
db > insert 2 voltorb volty@example.com
Executed.
db > .exit
~
~ ./db mydb.db
db > select
(1, cstack, foo@bar.com)
(2, voltorb, volty@example.com)
Executed.
db > .exit
~
```

Per divertimento extra, diamo un'occhiata a `mydb.db` per vedere come i nostri dati vengono memorizzati. Userò vim come editor esadecimale per guardare il layout di memoria del file:

```
vim mydb.db
:%!xxd
```
{% include image.html url="assets/images/file-format.png" description="Formato File Attuale" %}

I primi quattro byte sono l'id della prima riga (4 byte perché memorizziamo un `uint32_t`). È memorizzato in ordine byte little-endian, quindi il byte meno significativo viene prima (01), seguito dai byte di ordine superiore (00 00 00). Abbiamo usato `memcpy()` per copiare byte dalla nostra struct `Row` nella cache delle pagine, quindi questo significa che la struct era disposta in memoria in ordine byte little-endian. Questo è un attributo della macchina per cui ho compilato il programma. Se volessimo scrivere un file di database sulla mia macchina, poi leggerlo su una macchina big-endian, dovremmo cambiare i nostri metodi `serialize_row()` e `deserialize_row()` per sempre memorizzare e leggere byte nello stesso ordine.

I prossimi 33 byte memorizzano l'username come stringa terminata da null. Apparentemente "cstack" in ASCII esadecimale è `63 73 74 61 63 6b`, seguito da un carattere null (`00`). Il resto dei 33 byte non è usato.

I prossimi 256 byte memorizzano l'email nello stesso modo. Qui possiamo vedere alcuni dati casuali dopo il carattere null di terminazione. Questo è molto probabilmente dovuto a memoria non inizializzata nella nostra struct `Row`. Copiamo l'intero buffer email di 256 byte nel file, inclusi tutti i byte dopo la fine della stringa. Qualunque cosa fosse in memoria quando abbiamo allocato quella struct è ancora lì. Ma dato che usiamo un carattere null di terminazione, non ha effetto sul comportamento.

**NOTA**: Se volessimo assicurarci che tutti i byte siano inizializzati, sarebbe sufficiente usare `strncpy` invece di `memcpy` mentre copiamo i campi `username` ed `email` delle righe in `serialize_row`, così:

```diff
 void serialize_row(Row* source, void* destination) {
     memcpy(destination + ID_OFFSET, &(source->id), ID_SIZE);
-    memcpy(destination + USERNAME_OFFSET, &(source->username), USERNAME_SIZE);
-    memcpy(destination + EMAIL_OFFSET, &(source->email), EMAIL_SIZE);
+    strncpy(destination + USERNAME_OFFSET, source->username, USERNAME_SIZE);
+    strncpy(destination + EMAIL_OFFSET, source->email, EMAIL_SIZE);
 }
```

## Conclusione

Perfetto! Abbiamo la persistenza. Non è la migliore. Per esempio se uccidi il programma senza digitare `.exit`, perdi le tue modifiche. Inoltre, stiamo scrivendo tutte le pagine di nuovo su disco, anche pagine che non sono cambiate da quando le abbiamo lette dal disco. Questi sono problemi che possiamo affrontare più tardi.

La prossima volta introdurremo i cursori, che dovrebbero rendere più facile implementare il B-tree.

Fino ad allora!

## Diff Completo
```diff
+#include <errno.h>
+#include <fcntl.h>
 #include <stdbool.h>
 #include <stdio.h>
 #include <stdlib.h>
 #include <string.h>
 #include <stdint.h>
+#include <unistd.h>

 struct InputBuffer_t {
   char* buffer;
@@ -62,9 +65,16 @@ const uint32_t PAGE_SIZE = 4096;
 const uint32_t ROWS_PER_PAGE = PAGE_SIZE / ROW_SIZE;
 const uint32_t TABLE_MAX_ROWS = ROWS_PER_PAGE * TABLE_MAX_PAGES;

+typedef struct {
+  int file_descriptor;
+  uint32_t file_length;
+  void* pages[TABLE_MAX_PAGES];
+} Pager;
+
 typedef struct {
   uint32_t num_rows;
-  void* pages[TABLE_MAX_PAGES];
+  Pager* pager;
 } Table;

@@ -84,32 +94,81 @@ void deserialize_row(void *source, Row* destination) {
   memcpy(&(destination->email), source + EMAIL_OFFSET, EMAIL_SIZE);
 }

+void* get_page(Pager* pager, uint32_t page_num) {
+  if (page_num > TABLE_MAX_PAGES) {
+     printf("Tried to fetch page number out of bounds. %d > %d\n", page_num,
+     	TABLE_MAX_PAGES);
+     exit(EXIT_FAILURE);
+  }

  if (pager->pages[page_num] == NULL) {
     // Cache miss. Alloca memoria e carica dal file.
     void* page = malloc(PAGE_SIZE);
     uint32_t num_pages = pager->file_length / PAGE_SIZE;

     // Potremmo salvare una pagina parziale alla fine del file
     if (pager->file_length % PAGE_SIZE) {
         num_pages += 1;
     }

     if (page_num <= num_pages) {
         lseek(pager->file_descriptor, page_num * PAGE_SIZE, SEEK_SET);
         ssize_t bytes_read = read(pager->file_descriptor, page, PAGE_SIZE);
         if (bytes_read == -1) {
     	printf("Error reading file: %d\n", errno);
     	exit(EXIT_FAILURE);
         }
     }

     pager->pages[page_num] = page;
  }

  return pager->pages[page_num];
}

 void* row_slot(Table* table, uint32_t row_num) {
   uint32_t page_num = row_num / ROWS_PER_PAGE;
-  void *page = table->pages[page_num];
-  if (page == NULL) {
-     // Alloca memoria solo quando proviamo ad accedere alla pagina
-     page = table->pages[page_num] = malloc(PAGE_SIZE);
-  }
+  void *page = get_page(table->pager, page_num);
   uint32_t row_offset = row_num % ROWS_PER_PAGE;
   uint32_t byte_offset = row_offset * ROW_SIZE;
   return page + byte_offset;
 }

-Table* new_table() {
-  Table* table = malloc(sizeof(Table));
-  table->num_rows = 0;
+Pager* pager_open(const char* filename) {
+  int fd = open(filename,
+     	  O_RDWR | 	// Modalità Read/Write
+     	      O_CREAT,	// Crea file se non esiste
+     	  S_IWUSR |	// Permesso di scrittura utente
+     	      S_IRUSR	// Permesso di lettura utente
+     	  );

  if (fd == -1) {
     printf("Unable to open file\n");
     exit(EXIT_FAILURE);
  }

  off_t file_length = lseek(fd, 0, SEEK_END);

  Pager* pager = malloc(sizeof(Pager));
  pager->file_descriptor = fd;
  pager->file_length = file_length;

  for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++) {
-     table->pages[i] = i;
+     pager->pages[i] = NULL;
  }
-  return table;
+
  return pager;
}

-void free_table(Table* table) {
-  for (int i = 0; table->pages[i]; i++) {
-     free(table->pages[i]);
-  }
-  free(table);
+Table* db_open(const char* filename) {
+  Pager* pager = pager_open(filename);
+  uint32_t num_rows = pager->file_length / ROW_SIZE;

  Table* table = malloc(sizeof(Table));
  table->pager = pager;
  table->num_rows = num_rows;

  return table;
}

 InputBuffer* new_input_buffer() {
@@ -142,10 +201,76 @@ void close_input_buffer(InputBuffer* input_buffer) {
   free(input_buffer);
 }

+void pager_flush(Pager* pager, uint32_t page_num, uint32_t size) {
+  if (pager->pages[page_num] == NULL) {
+     printf("Tried to flush null page\n");
+     exit(EXIT_FAILURE);
+  }

  off_t offset = lseek(pager->file_descriptor, page_num * PAGE_SIZE,
     		 SEEK_SET);

  if (offset == -1) {
     printf("Error seeking: %d\n", errno);
     exit(EXIT_FAILURE);
  }

  ssize_t bytes_written = write(
      pager->file_descriptor, pager->pages[page_num], size
      );

  if (bytes_written == -1) {
     printf("Error writing: %d\n", errno);
     exit(EXIT_FAILURE);
  }
}

+void db_close(Table* table) {
+  Pager* pager = table->pager;
+  uint32_t num_full_pages = table->num_rows / ROWS_PER_PAGE;

  for (uint32_t i = 0; i < num_full_pages; i++) {
     if (pager->pages[i] == NULL) {
         continue;
     }
     pager_flush(pager, i, PAGE_SIZE);
     free(pager->pages[i]);
     pager->pages[i] = NULL;
  }

  // Potrebbe esserci una pagina parziale da scrivere alla fine del file
  // Questo non dovrebbe essere necessario dopo che passeremo a un B-tree
  uint32_t num_additional_rows = table->num_rows % ROWS_PER_PAGE;
  if (num_additional_rows > 0) {
     uint32_t page_num = num_full_pages;
     if (pager->pages[page_num] != NULL) {
         pager_flush(pager, page_num, num_additional_rows * ROW_SIZE);
         free(pager->pages[page_num]);
         pager->pages[page_num] = NULL;
     }
  }

  int result = close(pager->file_descriptor);
  if (result == -1) {
     printf("Error closing db file.\n");
     exit(EXIT_FAILURE);
  }
  for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++) {
     void* page = pager->pages[i];
     if (page) {
         free(page);
         pager->pages[i] = NULL;
     }
  }

  free(pager);
  free(table);
}

 MetaCommandResult do_meta_command(InputBuffer* input_buffer, Table *table) {
   if (strcmp(input_buffer->buffer, ".exit") == 0) {
     close_input_buffer(input_buffer);
     db_close(table);
     exit(EXIT_SUCCESS);
   } else {
     return META_COMMAND_UNRECOGNIZED_COMMAND;
```

PrepareResult prepare_insert(InputBuffer* input_buffer, Statement* statement) {
   if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
     statement->type = STATEMENT_INSERT;
     sscanf(input_buffer->buffer, "insert %d %s %s", &(statement->row_to_insert.id),
```
