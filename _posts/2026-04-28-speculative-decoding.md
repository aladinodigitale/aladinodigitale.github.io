---
title: "Speculative decoding: quando il problema non è solo leggere il prompt, ma anche generare la risposta"
date: 2026-04-28
tags: [AI locale, LLM, speculative decoding, llama.cpp, LM Studio, Qwen3.6, coding agent]
excerpt: "Lo speculative decoding accelera la generazione token per token usando un modello piccolo o pattern ricorrenti. Vediamo come funziona e se può aiutare con un modello denso come Qwen3.6 27B."
author: alessio
classes: wide
header:
  overlay_image: /assets/images/speculative-decoding/overlay-spec-dec.jpg
  overlay_filter: 0.4
---

Negli ultimi mesi si è parlato molto di modelli locali, coding agent e inferenza su hardware consumer. Una delle cose che ho sottolineato più volte è che, quando si usa un coding agent su una codebase reale, uno dei colli di bottiglia principali è spesso la fase di **prefill**.

Il modello deve leggere istruzioni, file, cronologia della conversazione, output dei tool, porzioni di codice e magari anche parti del progetto ripetute più volte nel contesto. In quei casi, il problema non è tanto "quanto velocemente scrive", ma "quanto costa fargli rileggere tutto".

Questo articolo, però, parla di un'altra fase della pipeline: la **generazione** vera e propria, cioè il momento in cui il modello produce la risposta token dopo token.

Le due cose non sono in contraddizione. Sono problemi diversi.

Il prefill si affronta con ottimizzazioni come riuso della KV-cache, prefix caching, gestione più intelligente del contesto, riduzione del prompt, attenzione più efficiente e, in alcuni casi, quantizzazione della KV-cache. Lo **speculative decoding**, invece, prova ad accelerare soprattutto la fase successiva: quella in cui il modello ha già letto il contesto e deve iniziare a scrivere.

E questa fase può diventare un problema, soprattutto quando si usano modelli locali densi, grandi e qualitativamente interessanti, ma non velocissimi in generazione.

Nel mio caso un modello da testare è sicuramente [*Qwen3.6 27B*](https://huggingface.co/Qwen/Qwen3.6-27B), denso, interessante per attività di coding, ma non esattamente un fulmine quando deve generare molti token su hardware locale. Ho una macchina basata su AMD Ryzen AI Max+ 395 con 128GB di memoria unificats, quindi posso caricare anche modelli relativamente importanti, ma questo non significa automaticamente avere una velocità di decode paragonabile a quella di un servizio cloud.

Da qui nasce l'interesse per lo **speculative decoding**, inteso come tassello nello stack di ottimizzazioni che rendono sempre più praticabile l'uso locale di modelli avanzati.

---

## L'idea dello speculative decoding

Lo speculative decoding è una famiglia di tecniche che prova ad accelerare la generazione facendo una cosa apparentemente semplice: qualcuno prova a indovinare i prossimi token, e il modello principale verifica se vanno bene.

La versione più intuitiva è questa:

1. un modello piccolo e veloce, chiamato spesso **draft model**, propone una sequenza di token;
2. il modello principale, più grande e più lento, controlla quella sequenza;
3. i token compatibili con quello che il modello principale avrebbe prodotto vengono accettati;
4. quando la previsione sbaglia, il modello principale riprende il controllo.

Il punto importante è che il modello piccolo non decide davvero la risposta. Propone soltanto una bozza. Il modello principale mantiene il diritto di veto.

Una formulazione semplice potrebbe essere:

> Il draft model non sostituisce il modello grande. Cerca solo di anticiparlo.

Se il draft model indovina spesso, il sistema può evitare parte del lavoro token-by-token e generare più velocemente. Se invece indovina poco, il vantaggio si riduce o sparisce.

La [documentazione di `llama.cpp`](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md) descrive proprio il draft model come l'approccio più comune allo speculative decoding: un modello molto più piccolo genera draft che vengono poi verificati dal modello principale. La stessa documentazione elenca anche implementazioni senza draft model, tra cui varianti basate su n-gram.

---

## Perché se ne parla proprio ora

Lo speculative decoding non è un'idea nata ieri. La cosa interessante è che sta diventando più accessibile negli strumenti che molte persone usano davvero per l'inferenza locale.

Da un lato ci sono applicazioni come [LM Studio](https://lmstudio.ai), che permettono di attivare speculative decoding in modo relativamente semplice e, cosa molto utile dal punto di vista didattico, mostrano visivamente i token proposti dal draft model e accettati dal modello principale. LM Studio ha documentato questa funzionalità spiegando che il draft model deve essere compatibile con il modello principale, in particolare lato vocabolario/tokenizer.

Dall'altro lato c'è **llama.cpp**, che nelle versioni recenti sta sperimentando varie forme di speculative decoding, incluse modalità con draft model e modalità n-gram. La [documentazione attuale di llama.cpp](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md) elenca più implementazioni per `llama-server`, tra cui `draft`, `ngram-cache`, `ngram-simple`, `ngram-map-k` e `ngram-mod`.

Questo è rilevante perché, con modelli recenti come Qwen3.5 e Qwen3.6, il supporto e le combinazioni praticabili sono in evoluzione rapida. Una [PR recente di llama.cpp](https://github.com/thc1006/qwen3.6-speculative-decoding-rtx3090) (#19493) ha ad esempio abilitato il draft-model speculative decoding per i modelli Qwen3.5/3.6 MoE (Mixture of Experts), aprendo la strada a nuovi esperimenti.

Questo spiega anche una differenza pratica importante: in **LM Studio** oggi la demo più semplice può richiedere coppie di modelli più vecchie o già supportate, mentre usando direttamente una versione recente di **llama.cpp** si possono fare esperimenti più avanzati con Qwen3.5/Qwen3.6.

LM Studio usa llama.cpp come backend in molti casi, ma non espone necessariamente subito ogni funzionalità appena arrivata upstream. Per questo ha senso fare due esempi separati: uno in LM Studio per capire visivamente il meccanismo, e uno in llama.cpp per un caso più vicino al mio workflow reale.

---

## Demo 1: vedere speculative decoding in LM Studio

La prima demo che voglio usare nell'articolo non serve a dimostrare il massimo speedup possibile. Serve a far vedere il concetto.

Il setup è questo:

* applicazione: **LM Studio**;
* modello principale: **Qwen 3 14B**;
* draft model: **Qwen 3 0.6B**; 
* speculative decoding abilitato;
* visualizzazione dei token accettati dal draft model attiva.

Chiaramente non scelgo questa coppia perché sia lo stato dell'arte per il coding locale. La scelgo perché, nel mio setup, è una delle combinazioni che LM Studio permette di usare facilmente per mostrare il meccanismo.

La cosa interessante è la visualizzazione: durante la generazione, LM Studio può evidenziare i token proposti dal draft model e accettati dal modello principale. Dal punto di vista didattico è molto efficace, perché trasforma un concetto un po' astratto in qualcosa che si vede davvero.

Vediamo dunque come la risposta ad un prompt direttamente nella console di LMStudio viene servita Qwen3 14B generando in media attorno ai 14 token/s.

{% include figure image_path="/assets/images/speculative-decoding/lmstudio-baseline.png" alt="" caption="LM Studio con speculative decoding non attivo." %}

Una volta attivato speculative decoding molti token (in verde qui sotto) vengono correttamente indovinati dal modello draft Qwen 3 0.6B, portando ad una generazione media attorno ai 24 token/s.

{% include figure image_path="/assets/images/speculative-decoding/lmstudio-spec-dec.png" alt="Chat LM Studio con token/s visibili" caption="Chat in LM Studio con speculative decoding attivo." %}

Quello che voglio mostrare è semplice:

* il draft model prova a generare token in anticipo;
* il modello principale li verifica;
* alcuni token vengono accettati;
* più token vengono accettati, più il meccanismo può diventare utile;
* quando il draft model non indovina, il vantaggio si riduce.

Questa demo è utile soprattutto per capire una cosa: speculative decoding non è "il modello piccolo che risponde al posto di quello grande". È più simile a un assistente che prova a completare le frasi, mentre il modello grande controlla che il completamento sia corretto.

---

## Il limite della demo in LM Studio

La demo in LM Studio è bella perché rende visibile il meccanismo. Ma non rappresenta necessariamente un caso d'uso reale attuale.

In particolare mi interessa soprattutto capire se queste tecniche possono aiutare quando uso un modello locale moderno e più pesante per attività di coding.

Uno scenario rappresentativo è dunque simile a questo:

* modello denso;
* buona qualità su codice;
* generazione non velocissima;
* task di refactoring o modifica di una codebase reale;
* output lungo abbastanza da rendere percepibile la differenza.

Per questo la seconda demo usa un approccio diverso: **Qwen3.6 27B** con llama.cpp e speculative decoding basato su **n-gram**.

---

## Draft model e n-gram: due strade diverse

Nel primo esempio abbiamo visto la forma più intuitiva di speculative decoding: un secondo modello, più piccolo, propone token.

Ma non è l'unica possibilità.

In llama.cpp esistono anche approcci "draftless", cioè senza un vero draft model separato. Tra questi ci sono le modalità basate su **n-gram**.

Un n-gram, semplificando, è una sequenza di token. L'idea è cercare sequenze già viste nel prompt o nella generazione corrente e usarle per proporre i token successivi.

Invece di chiedere a un modello piccolo "secondo te cosa viene dopo?", il sistema fa qualcosa di più meccanico:

> Ho già visto una sequenza simile nel contesto. Dopo quella sequenza, quali token venivano? Proviamo a proporli anche qui.

Questo approccio ha caratteristiche diverse dal draft model.

| Aspetto                                      | Draft model                                    | N-gram speculative decoding                               |
| -------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Usa un secondo modello                       | Sì                                             | Non necessariamente                                       |
| Richiede compatibilità tokenizer/vocabolario | Sì                                             | No, perché lavora sui token del modello principale        |
| Consuma memoria aggiuntiva                   | Sì                                             | Molto meno                                                |
| Capisce semanticamente il testo              | In parte, tramite il draft model               | No, sfrutta pattern e ripetizioni                         |
| Caso ideale                                  | Draft model piccolo ma ben allineato           | Testi ripetitivi, codice, refactoring, output strutturati |
| Rischio principale                           | Il draft model è troppo lento o poco allineato | Il testo non contiene pattern utili                       |

La documentazione di llama.cpp indica che le implementazioni con draft model possono essere combinate con implementazioni senza draft model, e mostra anche esempi in cui `ngram-mod` viene usato insieme a un draft model.

Per il mio secondo esempio, però, l'interesse principale è proprio valutare l'approccio n-gram, perché non richiede di caricare un altro modello e perché il codice è un dominio in cui la ripetizione è spesso molto forte.

---

## Perché n-gram ha senso nel codice

Nel testo creativo, ogni frase può essere nuova. In un refactoring di codice, ad esempio Java, invece, moltissime parti dell'output sono prevedibili o assomigliano a qualcosa che è già nel contesto.

Pensiamo a una classe Java reale. Dentro potremmo trovare:

* dichiarazioni di package;
* import;
* annotazioni;
* firme di metodi;
* nomi di classi già presenti;
* nomi di variabili ricorrenti;
* pattern di logging;
* strutture `try/catch`;
* chiamate a servizi;
* test simili;
* blocchi ripetuti con piccole variazioni.

Quando chiediamo a un modello di fare refactoring, spesso non gli stiamo chiedendo di inventare tutto da zero. Gli stiamo chiedendo di riconoscere una struttura, modificarla, ripeterla in modo coerente e adattarla.

Questo è esattamente il tipo di situazione in cui un approccio basato su n-gram può avere senso, non perché capisca il codice, ma perché il codice contiene molte sequenze che ritornano.

---

## Demo 2: refactoring Java con Qwen3.6 27B e llama.cpp

Ho dunque configurato Claude Code per utilizzare in inferenza un modello Qwen3.6 27B servito da una versione recentissima di llama.cpp, così da disporre delle ultime patch e poter attivare con successo n-gram speculative decoding.

Il task prevedeva di fare refactoring di una classe di diverse centinaia di righe di codice, deprecando tutti i metodi e delegando ad una nuova versione in cui la logica della classe stessa fosse mantenuta ma il codice fosse ri-organizzato e modernizzato con pattern più idiomatici. Altri task che si sarebbero prestati a questo test avrebbero potuto coinvolgere ad esempio:

* semplificazione di una classe troppo lunga;
* aggiunta o riorganizzazione di test;
* separazione di responsabilità tra metodi;
* estrazione di logica duplicata in metodi privati.

Ho eseguito lo stesso refactoring anche senza speculative decoding; lo scopo chiaramente era quello di capire se la tecnica permettesse veramente di recuperare del tempo in fase di generazione.


### Perché Qwen3.6 27B

Qwen3.6 27B è interessante perché rappresenta bene una categoria di modelli che, su AMD Strix Halo, sono allo stesso tempo desiderabili (essendo densi, ben si prestano a coding e ragionamento con agenti) e accettabili in fase di prefill ma estremamente lenti in generazione (di nuovo perché densi).

Desiderabili perché un modello denso di questa taglia può essere molto interessante per coding, ragionamento tecnico e agenti locali.
Quindi speculative deconding diventa una possibile soluzione per rendere più fluido l'utilizzo di un buon modello altrimenti inadatto.

### Esecuzione

Ho specificato questi parametri nel avviare llama-server

```text
--draft-min 4 --draft-max 16 --spec-ngram-size-n 12 --spec-type ngram-mod
```

in sostanza controllando le dimensioni dei gruppi di token da pescare (e provare ad usare) dalle generazioni precedenti e contesto.


Nei log ho visto varie statistiche sull'utilizzo di ngram-mod, in particolare verso la fine del test

```text
statistics ngram_mod: #calls(b,g,a) = 43 10790 883, #gen drafts = 884, #acc drafts = 883, #gen tokens = 13766, #acc tokens = 7096, dur(b,g,a) = 145.503, 393.013, 64.153 ms
```

Rispetto alla baseline senza speculative decoding, l'esecuzione ngram ha permesso di risparmiare circa 5 min a fronte degli oltre 27 richiesti dal sistema per completare il refactoring.

Va detto che in realtà il modello ha generato molto meno codice "ripetuto" di quanto ci si potesse aspettare perché Claude Code ha fatto uso intelligente dei tool di cui disponeva, ad esempio per effettuare sostituzioni di testo in modo deterministico e efficiente.

---

## Come leggere il risultato senza farsi ingannare

Speculative decoding è una tecnica facile da fraintendere.

Un errore comune è guardare solo quanti token sono stati accettati dal draft o dall'n-gram e concludere che allora il sistema è sicuramente più veloce. Non è detto.

Il risultato netto dipende da vari fattori:

* quanto costa generare i draft;
* quanto costa verificarli;
* quanto spesso vengono accettati;
* quanto è lungo l'output;
* quanto è ripetitivo il testo;
* quale parte della pipeline è davvero il collo di bottiglia;
* quanto è efficiente l'implementazione nel backend;
* che hardware stiamo usando.

Se il problema principale è il prefill, speculative decoding non risolve il problema principale.

Se il modello genera già molto velocemente, il guadagno percepito può essere marginale.

Se il testo è poco ripetitivo, n-gram può aiutare poco.

Se invece il modello è lento in generazione e deve produrre molto codice strutturato, allora anche un guadagno parziale può cambiare la percezione d'uso.

È importante anche ricordare che i risultati variano molto. Ci sono test recenti su Qwen3.6-35B-A3B (un modello MoE) con RTX 3090 in cui varie configurazioni di speculative decoding [non hanno prodotto uno speedup netto rispetto alla baseline](https://github.com/thc1006/qwen3.6-speculative-decoding-rtx3090/blob/master/README.md). È utile però notare che quel risultato riguarda un'architettura MoE, che ha dinamiche diverse rispetto a un modello denso come il 27B usato qui.

Questo non significa che speculative decoding "non funzioni". Significa che non è una funzione magica da attivare aspettandosi automaticamente il doppio dei token al secondo.
Nel mio caso il test ha mostrato come si possa ottenere un miglioramento con il modello Qwen3.6 27B, ma probabilmente non tale da preferirlo ad un genericamente ben più veloce modello MoE come il Qwen3.5 122B A10B.

Resta il fatto che la ricerca attorno queste tecniche di ottimizzazione è super interessante e non è da escludere che nuovi algoritmi ed implementazioni ribaltino le conclusioni derivate da queste prove.

