---
title: Parte 4 - I Nostri Primi Test (e Bug)
date: 2017-09-03
---

Abbiamo la capacità di inserire righe nel nostro database e di stampare tutte le righe. Prendiamoci un momento per testare quello che abbiamo finora.

Userò [rspec](http://rspec.info/) per scrivere i miei test perché mi è familiare, e la sintassi è abbastanza leggibile.

Definirò un breve helper per inviare una lista di comandi al nostro programma database e poi fare asserzioni sull'output:

```ruby
describe 'database' do
  def run_script(commands)
    raw_output = nil
    IO.popen("./db", "r+") do |pipe|
      commands.each do |command|
        pipe.puts command
      end

      pipe.close_write

      # Leggi l'intero output
      raw_output = pipe.gets(nil)
    end
    raw_output.split("\n")
  end

  it 'inserts and retrieves a row' do
    result = run_script([
      "insert 1 user1 person1@example.com",
      "select",
      ".exit",
    ])
    expect(result).to match_array([
      "db > Executed.",
      "db > (1, user1, person1@example.com)",
      "Executed.",
      "db > ",
    ])
  end
end
```

Questo semplice test si assicura che otteniamo quello che abbiamo messo dentro. E infatti passa:
```command-line
bundle exec rspec
.

Finished in 0.00871 seconds (files took 0.09506 seconds to load)
1 example, 0 failures
```

Ora è fattibile testare l'inserimento di un gran numero di righe nel database:
```ruby
it 'prints error message when table is full' do
  script = (1..1401).map do |i|
    "insert #{i} user#{i} person#{i}@example.com"
  end
  script << ".exit"
  result = run_script(script)
  expect(result[-2]).to eq('db > Error: Table full.')
end
```

Eseguendo i test di nuovo...
```command-line
bundle exec rspec
..

Finished in 0.01553 seconds (files took 0.08156 seconds to load)
2 examples, 0 failures
```

Perfetto, funziona! Il nostro db può contenere 1400 righe ora perché abbiamo impostato il numero massimo di pagine a 100, e 14 righe possono stare in una pagina.

Leggendo attraverso il codice che abbiamo finora, ho realizzato che potremmo non gestire correttamente la memorizzazione dei campi di testo. Facile da testare con questo esempio:
```ruby
it 'allows inserting strings that are the maximum length' do
  long_username = "a"*32
  long_email = "a"*255
  script = [
    "insert 1 #{long_username} #{long_email}",
    "select",
    ".exit",
  ]
  result = run_script(script)
  expect(result).to match_array([
    "db > Executed.",
    "db > (1, #{long_username}, #{long_email})",
    "Executed.",
    "db > ",
  ])
end
```

E il test fallisce!
```ruby
Failures:

  1) database allows inserting strings that are the maximum length
     Failure/Error: raw_output.split("\n")

     ArgumentError:
       invalid byte sequence in UTF-8
     # ./spec/main_spec.rb:14:in `split'
     # ./spec/main_spec.rb:14:in `run_script'
     # ./spec/main_spec.rb:48:in `block (2 levels) in <top (required)>'
```

Se lo proviamo noi stessi, vedremo che ci sono alcuni caratteri strani quando proviamo a stampare la riga. (Sto abbreviando le stringhe lunghe):
```command-line
db > insert 1 aaaaa... aaaaa...
Executed.
db > select
(1, aaaaa...aaa\, aaaaa...aaa\)
Executed.
db >
```

Cosa sta succedendo? Se dai un'occhiata alla nostra definizione di una Row, allociamo esattamente 32 byte per username ed esattamente 255 byte per email. Ma le [stringhe C](http://www.cprogramming.com/tutorial/c/lesson9.html) dovrebbero terminare con un carattere null, per cui non abbiamo allocato spazio. La soluzione è allocare un byte aggiuntivo:
```diff
 const uint32_t COLUMN_EMAIL_SIZE = 255;
 typedef struct {
   uint32_t id;
-  char username[COLUMN_USERNAME_SIZE];
-  char email[COLUMN_EMAIL_SIZE];
+  char username[COLUMN_USERNAME_SIZE + 1];
+  char email[COLUMN_EMAIL_SIZE + 1];
 } Row;
 ```

 E infatti questo lo risolve:
 ```ruby
 bundle exec rspec
...

Finished in 0.0188 seconds (files took 0.08516 seconds to load)
3 examples, 0 failures
```

Non dovremmo permettere l'inserimento di username o email che sono più lunghi della dimensione della colonna. La specifica per quello è così:
```ruby
it 'prints error message if strings are too long' do
  long_username = "a"*33
  long_email = "a"*256
  script = [
    "insert 1 #{long_username} #{long_email}",
    "select",
    ".exit",
  ]
  result = run_script(script)
  expect(result).to match_array([
    "db > String is too long.",
    "db > Executed.",
    "db > ",
  ])
end
```

Per fare questo dobbiamo aggiornare il nostro parser. Come promemoria, stiamo attualmente usando [scanf()](https://linux.die.net/man/3/scanf):
```c
if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
  statement->type = STATEMENT_INSERT;
  int args_assigned = sscanf(
      input_buffer->buffer, "insert %d %s %s", &(statement->row_to_insert.id),
      statement->row_to_insert.username, statement->row_to_insert.email);
  if (args_assigned < 3) {
    return PREPARE_SYNTAX_ERROR;
  }
  return PREPARE_SUCCESS;
}
```

Ma [scanf ha alcuni svantaggi](https://stackoverflow.com/questions/2430303/disadvantages-of-scanf). Se la stringa che sta leggendo è più grande del buffer in cui sta leggendo, causerà un buffer overflow e inizierà a scrivere in posti inaspettati. Vogliamo controllare la lunghezza di ogni stringa prima di copiarla in una struttura `Row`. E per fare quello, dobbiamo dividere l'input per spazi.

Userò [strtok()](http://www.cplusplus.com/reference/cstring/strtok/) per fare quello. Penso che sia più facile da capire se lo vedi in azione:

```diff
+PrepareResult prepare_insert(InputBuffer* input_buffer, Statement* statement) {
+  statement->type = STATEMENT_INSERT;
+
+  char* keyword = strtok(input_buffer->buffer, " ");
+  char* id_string = strtok(NULL, " ");
+  char* username = strtok(NULL, " ");
+  char* email = strtok(NULL, " ");
+
+  if (id_string == NULL || username == NULL || email == NULL) {
+    return PREPARE_SYNTAX_ERROR;
+  }
+
+  int id = atoi(id_string);
+  if (strlen(username) > COLUMN_USERNAME_SIZE) {
+    return PREPARE_STRING_TOO_LONG;
+  }
+  if (strlen(email) > COLUMN_EMAIL_SIZE) {
+    return PREPARE_STRING_TOO_LONG;
+  }
+
+  statement->row_to_insert.id = id;
+  strcpy(statement->row_to_insert.username, username);
+  strcpy(statement->row_to_insert.email, email);
+
+  return PREPARE_SUCCESS;
+}
+
 PrepareResult prepare_statement(InputBuffer* input_buffer,
                                 Statement* statement) {
   if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+    return prepare_insert(input_buffer, statement);
-    statement->type = STATEMENT_INSERT;
-    int args_assigned = sscanf(
-        input_buffer->buffer, "insert %d %s %s", &(statement->row_to_insert.id),
-        statement->row_to_insert.username, statement->row_to_insert.email);
-    if (args_assigned < 3) {
-      return PREPARE_SYNTAX_ERROR;
-    }
-    return PREPARE_SUCCESS;
   }
```

Chiamare `strtok` successivamente sul buffer di input lo divide in sottostringhe inserendo un carattere null ogni volta che raggiunge un delimitatore (spazio, nel nostro caso). Restituisce un puntatore all'inizio della sottostringa.

Possiamo chiamare [strlen()](http://www.cplusplus.com/reference/cstring/strlen/) su ogni valore di testo per vedere se è troppo lungo.

Possiamo gestire l'errore come facciamo per qualsiasi altro codice di errore:
```diff
 enum PrepareResult_t {
   PREPARE_SUCCESS,
+  PREPARE_STRING_TOO_LONG,
   PREPARE_SYNTAX_ERROR,
   PREPARE_UNRECOGNIZED_STATEMENT
 };
```
```diff
 switch (prepare_statement(input_buffer, &statement)) {
   case (PREPARE_SUCCESS):
     break;
+  case (PREPARE_STRING_TOO_LONG):
+    printf("String is too long.\n");
+    continue;
   case (PREPARE_SYNTAX_ERROR):
     printf("Syntax error. Could not parse statement.\n");
     continue;
```

Il che fa passare il nostro test
```command-line
bundle exec rspec
....

Finished in 0.02284 seconds (files took 0.116 seconds to load)
4 examples, 0 failures
```

Mentre siamo qui, potremmo anche gestire un altro caso di errore:
```ruby
it 'prints an error message if id is negative' do
  script = [
    "insert -1 cstack foo@bar.com",
    "select",
    ".exit",
  ]
  result = run_script(script)
  expect(result).to match_array([
    "db > ID must be positive.",
    "db > Executed.",
    "db > ",
  ])
end
```
```diff
 enum PrepareResult_t {
   PREPARE_SUCCESS,
+  PREPARE_NEGATIVE_ID,
   PREPARE_STRING_TOO_LONG,
   PREPARE_SYNTAX_ERROR,
   PREPARE_UNRECOGNIZED_STATEMENT
@@ -148,9 +147,6 @@ PrepareResult prepare_insert(InputBuffer* input_buffer, Statement* statement) {
   }

   int id = atoi(id_string);
+  if (id < 0) {
+    return PREPARE_NEGATIVE_ID;
+  }
   if (strlen(username) > COLUMN_USERNAME_SIZE) {
     return PREPARE_STRING_TOO_LONG;
   }
@@ -230,9 +226,6 @@ int main(int argc, char* argv[]) {
     switch (prepare_statement(input_buffer, &statement)) {
       case (PREPARE_SUCCESS):
         break;
+      case (PREPARE_NEGATIVE_ID):
+        printf("ID must be positive.\n");
+        continue;
       case (PREPARE_STRING_TOO_LONG):
         printf("String is too long.\n");
         continue;
```

Perfetto, abbastanza test per ora. Il prossimo è una funzionalità molto importante: la persistenza! Salveremo il nostro database in un file e lo leggeremo di nuovo.

Sarà fantastico.

Ecco il diff completo per questa parte:
```diff
@@ -22,6 +22,8 @@

 enum PrepareResult_t {
   PREPARE_SUCCESS,
+  PREPARE_NEGATIVE_ID,
+  PREPARE_STRING_TOO_LONG,
   PREPARE_SYNTAX_ERROR,
   PREPARE_UNRECOGNIZED_STATEMENT
  };
@@ -34,8 +36,8 @@
 #define COLUMN_EMAIL_SIZE 255
 typedef struct {
   uint32_t id;
-  char username[COLUMN_USERNAME_SIZE];
-  char email[COLUMN_EMAIL_SIZE];
+  char username[COLUMN_USERNAME_SIZE + 1];
+  char email[COLUMN_EMAIL_SIZE + 1];
 } Row;

@@ -150,18 +152,40 @@ MetaCommandResult do_meta_command(InputBuffer* input_buffer, Table *table) {
   }
 }

-PrepareResult prepare_statement(InputBuffer* input_buffer,
-                                Statement* statement) {
-  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+PrepareResult prepare_insert(InputBuffer* input_buffer, Statement* statement) {
   statement->type = STATEMENT_INSERT;
-  int args_assigned = sscanf(
-     input_buffer->buffer, "insert %d %d %s", &(statement->row_to_insert.id),
-     statement->row_to_insert.username, statement->row_to_insert.email
-     );
-  if (args_assigned < 3) {
+  char* keyword = strtok(input_buffer->buffer, " ");
+  char* id_string = strtok(NULL, " ");
+  char* username = strtok(NULL, " ");
+  char* email = strtok(NULL, " ");
+
+  if (id_string == NULL || username == NULL || email == NULL) {
      return PREPARE_SYNTAX_ERROR;
   }
+
+  int id = atoi(id_string);
+  if (id < 0) {
+     return PREPARE_NEGATIVE_ID;
+  }
+  if (strlen(username) > COLUMN_USERNAME_SIZE) {
+     return PREPARE_STRING_TOO_LONG;
+  }
+  if (strlen(email) > COLUMN_EMAIL_SIZE) {
+     return PREPARE_STRING_TOO_LONG;
+  }
+
+  statement->row_to_insert.id = id;
+  strcpy(statement->row_to_insert.username, username);
+  strcpy(statement->row_to_insert.email, email);
+
   return PREPARE_SUCCESS;
+
+}
+PrepareResult prepare_statement(InputBuffer* input_buffer,
+                                Statement* statement) {
+  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+      return prepare_insert(input_buffer, statement);
   }
   if (strcmp(input_buffer->buffer, "select") == 0) {
     statement->type = STATEMENT_SELECT;
@@ -223,6 +247,12 @@ int main(int argc, char* argv[]) {
     switch (prepare_statement(input_buffer, &statement)) {
       case (PREPARE_SUCCESS):
         break;
+      case (PREPARE_NEGATIVE_ID):
+	printf("ID must be positive.\n");
+	continue;
+      case (PREPARE_STRING_TOO_LONG):
+	printf("String is too long.\n");
+	continue;
       case (PREPARE_SYNTAX_ERROR):
 	printf("Syntax error. Could not parse statement.\n");
 	continue;
```
E abbiamo aggiunto test:
```diff
+describe 'database' do
+  def run_script(commands)
+    raw_output = nil
+    IO.popen("./db", "r+") do |pipe|
+      commands.each do |command|
+        pipe.puts command
+      end
+
+      pipe.close_write
+
+      # Leggi l'intero output
+      raw_output = pipe.gets(nil)
+    end
+    raw_output.split("\n")
+  end
+
+  it 'inserts and retrieves a row' do
+    result = run_script([
+      "insert 1 user1 person1@example.com",
+      "select",
+      ".exit",
+    ])
+    expect(result).to match_array([
+      "db > Executed.",
+      "db > (1, user1, person1@example.com)",
+      "Executed.",
+      "db > ",
+    ])
+  end
+
+  it 'prints error message when table is full' do
+    script = (1..1401).map do |i|
+      "insert #{i} user#{i} person#{i}@example.com"
+    end
+    script << ".exit"
+    result = run_script(script)
+    expect(result[-2]).to eq('db > Error: Table full.')
+  end
+
+  it 'allows inserting strings that are the maximum length' do
+    long_username = "a"*32
+    long_email = "a"*255
+    script = [
+      "insert 1 #{long_username} #{long_email}",
+      "select",
+      ".exit",
+    ]
+    result = run_script(script)
+    expect(result).to match_array([
+      "db > Executed.",
+      "db > (1, #{long_username}, #{long_email})",
+      "Executed.",
+      "db > ",
+    ])
+  end
+
+  it 'prints error message if strings are too long' do
+    long_username = "a"*33
+    long_email = "a"*256
+    script = [
+      "insert 1 #{long_username} #{long_email}",
+      "select",
+      ".exit",
+    ]
+    result = run_script(script)
+    expect(result).to match_array([
+      "db > String is too long.",
+      "db > Executed.",
+      "db > ",
+    ])
+  end
+
+  it 'prints an error message if id is negative' do
+    script = [
+      "insert -1 cstack foo@bar.com",
+      "select",
+      ".exit",
+    ]
+    result = run_script(script)
+    expect(result).to match_array([
+      "db > ID must be positive.",
+      "db > Executed.",
+      "db > ",
+    ])
+  end
+end
```
