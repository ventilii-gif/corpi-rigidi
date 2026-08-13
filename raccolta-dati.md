# Raccolta dati dei quiz → Foglio Google (Apps Script)

La webapp può inviare, al termine di ogni quiz e del test finale, un **report**
con: nome e cognome dello studente, i quesiti proposti con la risposta data
(giusta/sbagliata e tempo impiegato per ciascuna) e il tempo di permanenza in
ogni sotto-scheda. I dati finiscono in un **Foglio Google**, una riga per invio.

## 1. Crea il Foglio e lo script

1. Crea un nuovo **Foglio Google** (sarà il registro delle risposte).
2. Menu **Estensioni → Apps Script**.
3. Cancella il contenuto e incolla il codice qui sotto (`Codice.gs`).
4. Salva (💾).

```javascript
// Riceve i report della webapp "Corpi rigidi in rotazione" e li scrive nel Foglio.
function doPost(e){
  var lock = LockService.getScriptLock();
  lock.waitLock(30000);
  try{
    var data = JSON.parse(e.postData.contents);
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var sh = ss.getSheetByName('Risposte') || ss.insertSheet('Risposte');
    if (sh.getLastRow() === 0){
      sh.appendRow(['Data/ora','Nome e Cognome','Quiz','Punteggio',
                    'Tempo totale (s)','Tempi sotto-schede','Dettaglio quesiti','Lingua','JSON']);
      sh.setFrozenRows(1);
    }
    sh.appendRow([
      new Date(),
      data.name || '',
      data.quiz || '',
      (data.score != null ? data.score + '/' + data.total : ''),
      data.dwellTotalSec || '',
      data.dwellText || '',
      data.answersText || '',
      data.lang || '',
      JSON.stringify(data)
    ]);
    return ContentService.createTextOutput(JSON.stringify({ok:true}))
                         .setMimeType(ContentService.MimeType.JSON);
  } catch(err){
    return ContentService.createTextOutput(JSON.stringify({ok:false, error:String(err)}))
                         .setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}

function doGet(){
  return ContentService.createTextOutput('Endpoint attivo: la webapp invia i dati in POST.');
}
```

## 2. Pubblica come App web

1. In Apps Script: **Distribuisci → Nuova distribuzione**.
2. Icona ingranaggio → **Tipo: App web**.
3. Imposta:
   - **Esegui come:** _Me stesso_ (il tuo account: così lo script può scrivere nel foglio);
   - **Chi ha accesso:** _Chiunque_ (necessario perché gli studenti inviino senza login Google).
4. **Distribuisci** e autorizza quando richiesto.
5. Copia l'**URL dell'App web** (finisce con `/exec`).

## 3. Collega la webapp

Apri `index.html` **e** `en/index.html` e incolla l'URL nella costante, in cima al blocco
«Raccolta dati»:

```javascript
const REPORT_ENDPOINT='https://script.google.com/macros/s/.../exec';
```

Fatto: al termine di ogni quiz e del test finale comparirà il tasto verde
**«📤 Invia report all'insegnante»**, che scrive una riga nel foglio.

> Se preferisci, mandami tu l'URL e lo incollo io nei due file.

## Note tecniche

- L'invio usa `fetch(..., {mode:'no-cors'})`: il browser non può leggere la
  risposta (Apps Script non espone header CORS), ma la richiesta **arriva** e la
  riga viene scritta. Per questo il messaggio «✓ Report inviato» è ottimistico:
  la conferma vera è la comparsa della riga nel foglio.
- I tempi sono in **secondi**. «Tempi sotto-schede» elenca **tutte** le
  sotto-schede visitate nella sessione (Teoria, Simulazione, Esercizi, Quiz di
  ogni argomento, Home, Test finale). Il tempo con scheda in secondo piano
  (finestra nascosta) non viene conteggiato.
- «Dettaglio quesiti» riporta, per ogni quesito: esito (✓/✗), tempo, testo del
  quesito, risposta data e — se sbagliata — quella corretta.
- La colonna **JSON** contiene il payload completo, utile per analisi avanzate.
- Nessuna «parola d'ordine»: chiunque abbia il link della webapp può inviare.
  Se in futuro vuoi filtrare invii non autorizzati, si può aggiungere un codice
  classe condiviso.
