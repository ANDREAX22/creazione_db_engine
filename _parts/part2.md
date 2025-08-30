---
title: Parte 2 - Il Compilatore SQL e la Macchina Virtuale Più Semplici del Mondo
date: 2017-08-31
---

Stiamo facendo un clone di sqlite. Il "front-end" di sqlite è un compilatore SQL che analizza una stringa e produce una rappresentazione interna chiamata bytecode.

Questo bytecode viene passato alla macchina virtuale, che lo esegue.

{% include image.html url="assets/images/arch2.gif" description="Architettura SQLite (https://www.sqlite.org/arch.html)" %}

Dividere le cose in due passaggi come questo ha alcuni vantaggi:
- Riduce la complessità di ogni parte (es. la macchina virtuale non si preoccupa degli errori di sintassi)
- Permette di compilare query comuni una volta e memorizzare nella cache il bytecode per migliorare le prestazioni

Con questo in mente, rifattorizziamo la nostra funzione `main` e supportiamo due nuove parole chiave nel processo:

```diff
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);

-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```

Le istruzioni non-SQL come `.exit` sono chiamate "meta-comandi". Iniziano tutte con un punto, quindi le controlliamo e le gestiamo in una funzione separata.

Poi, aggiungiamo un passaggio che converte la riga di input nella nostra rappresentazione interna di un'istruzione. Questa è la nostra versione approssimativa del front-end di sqlite.

Infine, passiamo l'istruzione preparata a `execute_statement`. Questa funzione diventerà alla fine la nostra macchina virtuale.

Nota che due delle nostre nuove funzioni restituiscono enum che indicano successo o fallimento:

```c
typedef enum {
  META_COMMAND_SUCCESS,
  META_COMMAND_UNRECOGNIZED_COMMAND
} MetaCommandResult;

typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
```

"Istruzione non riconosciuta"? Sembra un po' come un'eccezione. Preferisco non usare eccezioni (e C non le supporta nemmeno), quindi sto usando codici di risultato enum ovunque sia pratico. Il compilatore C si lamenterà se il mio switch statement non gestisce un membro dell'enum, quindi possiamo sentirci un po' più sicuri di gestire ogni risultato di una funzione. Aspettati che più codici di risultato vengano aggiunti in futuro.

`do_meta_command` è solo un wrapper per la funzionalità esistente che lascia spazio per più comandi:

```c
MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
  if (strcmp(input_buffer->buffer, ".exit") == 0) {
    exit(EXIT_SUCCESS);
  } else {
    return META_COMMAND_UNRECOGNIZED_COMMAND;
  }
}
```

La nostra "istruzione preparata" ora contiene solo un enum con due possibili valori. Conterrà più dati quando permetteremo parametri nelle istruzioni:

```c
typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;

typedef struct {
  StatementType type;
} Statement;
```

`prepare_statement` (il nostro "Compilatore SQL") non capisce SQL ora. In effetti, capisce solo due parole:
```c
PrepareResult prepare_statement(InputBuffer* input_buffer,
                                Statement* statement) {
  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
    statement->type = STATEMENT_INSERT;
    return PREPARE_SUCCESS;
  }
  if (strcmp(input_buffer->buffer, "select") == 0) {
    statement->type = STATEMENT_SELECT;
    return PREPARE_SUCCESS;
  }

  return PREPARE_UNRECOGNIZED_STATEMENT;
}
```

Nota che usiamo `strncmp` per "insert" dato che la parola chiave "insert" sarà seguita da dati. (es. `insert 1 cstack foo@bar.com`)

Infine, `execute_statement` contiene alcuni stub:
```c
void execute_statement(Statement* statement) {
  switch (statement->type) {
    case (STATEMENT_INSERT):
      printf("This is where we would do an insert.\n");
      break;
    case (STATEMENT_SELECT):
      printf("This is where we would do a select.\n");
      break;
  }
}
```

Nota che non restituisce codici di errore perché non c'è ancora nulla che possa andare storto.

Con queste rifattorizzazioni, ora riconosciamo due nuove parole chiave!
```command-line
~ ./db
db > insert foo bar
This is where we would do an insert.
Executed.
db > delete foo
Unrecognized keyword at start of 'delete foo'.
db > select
This is where we would do a select.
Executed.
db > .tables
Unrecognized command '.tables'
db > .exit
~
```

Lo scheletro del nostro database sta prendendo forma... non sarebbe bello se memorizzasse dati? Nella prossima parte, implementeremo `insert` e `select`, creando il peggiore data store del mondo. Nel frattempo, ecco l'intero diff di questa parte:

```diff
@@ -10,6 +10,23 @@ struct InputBuffer_t {
 } InputBuffer;
 
+typedef enum {
+  META_COMMAND_SUCCESS,
+  META_COMMAND_UNRECOGNIZED_COMMAND
+} MetaCommandResult;
+
+typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
+
+typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;
+
+typedef struct {
+  StatementType type;
+} Statement;
+
 InputBuffer* new_input_buffer() {
   InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
   input_buffer->buffer = NULL;
@@ -40,17 +57,67 @@ void close_input_buffer(InputBuffer* input_buffer) {
     free(input_buffer);
 }
 
+MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
+  if (strcmp(input_buffer->buffer, ".exit") == 0) {
+    close_input_buffer(input_buffer);
+    exit(EXIT_SUCCESS);
+  } else {
+    return META_COMMAND_UNRECOGNIZED_COMMAND;
+  }
+}
+
+PrepareResult prepare_statement(InputBuffer* input_buffer,
+                                Statement* statement) {
+  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+    statement->type = STATEMENT_INSERT;
+    return PREPARE_SUCCESS;
+  }
+  if (strcmp(input_buffer->buffer, "select") == 0) {
+    statement->type = STATEMENT_SELECT;
+    return PREPARE_SUCCESS;
+  }
+
+  return PREPARE_UNRECOGNIZED_STATEMENT;
+}
+
+void execute_statement(Statement* statement) {
+  switch (statement->type) {
+    case (STATEMENT_INSERT):
+      printf("This is where we would do an insert.\n");
+      break;
+    case (STATEMENT_SELECT):
+      printf("This is where we would do a select.\n");
+      break;
+  }
+}
+
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);
 
-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```
