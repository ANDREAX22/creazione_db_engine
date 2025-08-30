---
title: Parte 1 - Introduzione e Configurazione del REPL
date: 2017-08-30
---

Come sviluppatore web, uso database relazionali ogni giorno al lavoro, ma sono una scatola nera per me. Alcune domande che ho:
- In che formato vengono salvati i dati? (in memoria e su disco)
- Quando si spostano dalla memoria al disco?
- Perché può esserci solo una chiave primaria per tabella?
- Come funziona il rollback di una transazione?
- Come sono formattati gli indici?
- Quando e come avviene una scansione completa della tabella?
- In che formato viene salvata una prepared statement?

In altre parole, come **funziona** un database?

Per capire le cose, sto scrivendo un database da zero. È modellato su sqlite perché è progettato per essere piccolo con meno funzionalità di MySQL o PostgreSQL, quindi ho una migliore speranza di capirlo. L'intero database è memorizzato in un singolo file!

# Sqlite

Ci sono molte [documentazioni degli interni di sqlite](https://www.sqlite.org/arch.html) sul loro sito web, più ho una copia di [SQLite Database System: Design and Implementation](https://play.google.com/store/books/details?id=9Z6IQQnX1JEC).

{% include image.html url="assets/images/arch1.gif" description="architettura sqlite (https://www.sqlite.org/zipvfs/doc/trunk/www/howitworks.wiki)" %}

Una query attraversa una catena di componenti per recuperare o modificare i dati. Il **front-end** consiste di:
- tokenizer
- parser
- code generator

L'input al front-end è una query SQL. l'output è bytecode della macchina virtuale sqlite (essenzialmente un programma compilato che può operare sul database).

Il _back-end_ consiste di:
- macchina virtuale
- B-tree
- pager
- interfaccia os

La **macchina virtuale** prende il bytecode generato dal front-end come istruzioni. Può quindi eseguire operazioni su una o più tabelle o indici, ognuno dei quali è memorizzato in una struttura dati chiamata B-tree. La VM è essenzialmente un grande switch statement sul tipo di istruzione bytecode.

Ogni **B-tree** consiste di molti nodi. Ogni nodo è lungo una pagina. Il B-tree può recuperare una pagina dal disco o salvarla di nuovo su disco emettendo comandi al pager.

Il **pager** riceve comandi per leggere o scrivere pagine di dati. È responsabile per la lettura/scrittura agli offset appropriati nel file del database. Mantiene anche una cache di pagine recentemente accessibili in memoria, e determina quando quelle pagine devono essere scritte di nuovo su disco.

L'**interfaccia os** è il livello che differisce a seconda del sistema operativo per cui sqlite è stato compilato. In questo tutorial, non supporterò piattaforme multiple.

[Un viaggio di mille miglia inizia con un singolo passo](https://en.wiktionary.org/wiki/a_journey_of_a_thousand_miles_begins_with_a_single_step), quindi iniziamo con qualcosa di un po' più semplice: il REPL.

## Creare un REPL Semplice

Sqlite avvia un ciclo read-execute-print quando lo avvii dalla riga di comando:

```shell
~ sqlite3
SQLite version 3.16.0 2016-11-04 19:09:39
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> create table users (id int, username varchar(255), email varchar(255));
sqlite> .tables
users
sqlite> .exit
~
```

Per fare questo, la nostra funzione main avrà un ciclo infinito che stampa il prompt, ottiene una riga di input, poi elabora quella riga di input:

```c
int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```

Definiremo `InputBuffer` come un piccolo wrapper intorno allo stato che dobbiamo memorizzare per interagire con [getline()](http://man7.org/linux/man-pages/man3/getline.3.html). (Ne parleremo di più tra un minuto)
```c
typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = (InputBuffer*)malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}
```

Poi, `print_prompt()` stampa un prompt all'utente. Lo facciamo prima di leggere ogni riga di input.

```c
void print_prompt() { printf("db > "); }
```

Per leggere una riga di input, usa [getline()](http://man7.org/linux/man-pages/man3/getline.3.html):
```c
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```
`lineptr` : un puntatore alla variabile che usiamo per puntare al buffer contenente la riga letta. Se è impostato su `NULL` viene allocato da `getline` e dovrebbe quindi essere liberato dall'utente, anche se il comando fallisce.

`n` : un puntatore alla variabile che usiamo per salvare la dimensione del buffer allocato.

`stream` : il flusso di input da cui leggere. Leggeremo dall'input standard.

`return value` : il numero di byte letti, che può essere inferiore alla dimensione del buffer.

Diciamo a `getline` di memorizzare la riga letta in `input_buffer->buffer` e la dimensione del buffer allocato in `input_buffer->buffer_length`. Memorizziamo il valore di ritorno in `input_buffer->input_length`.

`buffer` inizia come null, quindi `getline` alloca abbastanza memoria per contenere la riga di input e fa puntare `buffer` ad essa.

```c
void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}
```

Ora è appropriato definire una funzione che libera la memoria allocata per un'istanza di `InputBuffer *` e l'elemento `buffer` della rispettiva struttura (`getline` alloca memoria per `input_buffer->buffer` in `read_input`).

```c
void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}
```

Infine, analizziamo ed eseguiamo il comando. C'è solo un comando riconosciuto ora : `.exit`, che termina il programma. Altrimenti stampiamo un messaggio di errore e continuiamo il ciclo.

```c
if (strcmp(input_buffer->buffer, ".exit") == 0) {
  close_input_buffer(input_buffer);
  exit(EXIT_SUCCESS);
} else {
  printf("Unrecognized command '%s'.\n", input_buffer->buffer);
}
```

Proviamolo!
```shell
~ ./db
db > .tables
Unrecognized command '.tables'.
db > .exit
~
```

Perfetto, abbiamo un REPL funzionante. Nella prossima parte, inizieremo a sviluppare il nostro linguaggio di comando. Nel frattempo, ecco l'intero programma di questa parte:

```c
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}

void print_prompt() { printf("db > "); }

void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}

void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}

int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```
