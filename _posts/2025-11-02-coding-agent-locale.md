---
title: "Coding agent in locale: esperimenti con Claude Code Router e LM Studio"
date: 2025-11-02
excerpt: "Un esperimento pratico: far lavorare un coding agent interamente in locale e scoprire, tra cache, memoria e tempi di risposta, quanto sia davvero impegnativo."
tags: [AI locale, LLM, aiopenweight, Strix Halo, generativeai]
author: alessio
classes: wide
header:
  overlay_image: /assets/images/coding-agent-locale/overlay-image.jpg
  overlay_filter: 0.4
---

Durante la settimana appena conclusa ho cercato di capire fin dove si possa spingere l’idea di un *coding agent* — ma non nel cloud, bensì **in locale**, usando modelli openweight e il mio hardware.
Nel mio caso, si tratta di un **AMD Ryzen AI Max+ 395 (Strix Halo)** con **128 GB di RAM condivisa**: un sistema che sulla carta può gestire modelli anche piuttosto grandi, ma che nella pratica obbliga a ragionare su dettagli che, in cloud, di solito restano invisibili.

L’obiettivo era provare a replicare — almeno in parte — l’esperienza di **[Claude Code](https://www.claude.com/product/claude-code)**, l’ambiente di _Anthropic_ per lo sviluppo assistito da AI, ma eseguendo **tutto localmente**.

Per farlo mi sono servito di due componenti principali:

* **[Claude Code Router](https://github.com/musistudio/claude-code-router)**
* **[LM Studio](https://lmstudio.ai)**

Come usecase di test, ho scelto di "proseguire lo sviluppo" del piccolo progetto opensource ([onnx-model-generator-docker](https://github.com/asoldano/onnx-model-generator-docker)) che già avevo usato un mese fa proprio per [dimostrare le potenzialità di Claude Code](https://aladinodigitale.it/2025/10/01/claude-code-case-study.html). 

---

## Come funziona un coding agent

Prima di entrare nei dettagli, vale la pena ricordare cosa fa a grandi linee internamente un “coding agent”.
A differenza di una chat tradizionale, un agente non risponde solo a un prompt, ma **gestisce un ciclo di lavoro**:
pianifica un’azione (“implementa la feature X”, "risolvi in bug Y"), analizza il codice esistente, modifica file, richiama tool, legge risultati dei test e decide cosa fare dopo.

Ogni passo di questo processo genera **una o più richieste verso un modello linguistico**, che deve ragionare a partire da un contesto aggiornato.
Il modello, tuttavia, è **stateless**: non conserva memoria delle conversazioni e delle chiamate che le compongono.
Questo significa che *ogni volta* bisogna ridargli tutto ciò che serve — parte del codice, descrizioni, risultati intermedi — entro i limiti della **finestra di contesto**.

È proprio per questo che i coding agent necessitano di modelli con **contesti ampi (es. 64 k token o più)**: devono mantenere visibili porzioni significative di codice e output, altrimenti perdono “il filo” tra una richiesta e l’altra.

---

## Claude Code Router: il cuore dell’esperimento

Il *Claude Code Router* è un progetto open source pensato per reindirizzare le chiamate di Claude Code verso differenti modelli.
Non è un editor o un’interfaccia grafica: è un **proxy intelligente** che si interpone tra il client (CLI di Anthropic) e uno o più modelli configurati come backend.

Il router riceve le chiamate del client (ad esempio “pensa al prossimo passo”, “leggi questo file”, “genera test”), le instrada al modello corretto in base al tipo di richiesta (`think`, `longContext`, `default`, `background`), e restituisce la risposta nel formato previsto.

Ogni ruolo può usare un **modello diverso**, o la stessa istanza con parametri differenti — ad esempio uno piccolo per richieste rapide e uno grande per reasoning più complesso.
Nel mio caso, l’ho usato inizialmente con una sola istanza locale di **LM Studio**, poi in seguito con due (ma ci arrivo dopo).

---

## LM Studio: il backend locale

*LM Studio* è un’applicazione che consente di scaricare ed eseguire modelli LLM openweight su GPU o CPU locali, esponendoli tramite un’API compatibile con OpenAI.
Per il router è del tutto trasparente: crede di parlare con un endpoint remoto (come quello di Anthropic), ma in realtà l’inferenza avviene interamente sul mio sistema.

Ho scelto come modello di partenza **[gpt-oss-20b](https://huggingface.co/openai/gpt-oss-20b)**, che rappresenta un buon compromesso fra capacità e requisiti di memoria.
Le impostazioni iniziali in LM Studio erano:

* **Contesto massimo:** 64 k token
* **Flash Attention:** abilitata
* **Layer in GPU:** tutti

---

## 1. Prima configurazione: un solo modello per tutto

In questa prima prova, tutte le richieste del router (`default`, `think`, `background`, `longContext`) puntavano alla stessa istanza LM Studio con `gpt-oss-20b`.

```json
{
  ...
  "Providers": [
    {
      "name": "strix-halo",
      "api_base_url": "http://localhost:1234/v1/chat/completions",
      "api_key": "test",
      "models": [
        "openai/gpt-oss-20b"
      ]
    }
  ],

  "Router": {
    "default": "strix-halo,openai/gpt-oss-20b",
    "background": "strix-halo,openai/gpt-oss-20b",
    "think": "strix-halo,openai/gpt-oss-20b",
    "longContext": "strix-halo,openai/gpt-oss-20b",
    "longContextThreshold": 60000,
    "webSearch": "strix-halo,openai/gpt-oss-20b",
    "image": "strix-halo,openai/gpt-oss-20b"
  }
  ...
}
```

Nelle prime prove non avevo impostato limiti particolari per la risposta (`max_tokens`), lasciando al router il default.
Solo più avanti, con la configurazione multi-modello, il parametro `max_tokens` diventerà importante — ma non tanto per controllare la lunghezza dell’output, quanto per **come interagisce con la KV cache**, e quindi con la memoria effettiva utilizzata (ci torno tra poco).


### Perché 64 k token

Come accennato prima, un coding agent è *stateless*, quindi a ogni passo deve ricevere di nuovo tutto ciò che serve per ragionare: parte del codice del progetto, i file generati in precedenza, e le istruzioni aggiornate.
Un contesto da **64 k token** permette di contenere già una porzione significativa del codice e di mantenere coerenti le conversazioni multi-step.
Al di sotto dei 32 k, il modello “perde pezzi” del ragionamento, perché il contesto non basta a mantenere tutto il necessario.

In termini pratici, 64 k token equivalgono a circa 50 KB di testo grezzo (più metadati) e comportano un **overhead di memoria di circa 3 GB** solo per la gestione della cache associata a quel contesto.

### Layer in GPU e Flash Attention

Grazie all'abbondante memoria unificata del Ryzen AI Max+, posso abilitare *tutti i layer in GPU* ed eseguire l’intera pipeline del modello (i layer del transformer) direttamente sulla GPU, ottimizzando la velocità dell’inferenza e riducendo la latenza. Su un'architettura con scheda video dedicata, questo implicherebbe utilizzo della (spesso poca) memoria video VRAM, che diventa dunque l'elemento vincolante per capire quanti layer possono andare in GPU.

La **Flash Attention**, invece, è un’ottimizzazione della computazione delle matrici di attenzione: riduce la duplicazione di dati in memoria e accelera le moltiplicazioni tra `Q`, `K` e `V`.

{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-gpt-oss-20b-setup.png" alt="gpt-oss-20b setup" caption="Configurazione di LM Studio per avviare un server che esegua inferenza usando gpt-oss 20B." %}


---

### Prime osservazioni

Con questa configurazione base, l’agente funzionava ma in modo piuttosto lento.
Il tempo complessivo per ogni richiesta era dominato non tanto dalla velocità di generazione (token / s), quanto dal tempo di elaborazione del prompt iniziale.
Osservando i log di LM Studio, notavo che per diverse richieste inviate dal router il modello doveva ricostruire da zero tutto il contesto — e questo è esattamente il punto in cui mi sono dovuto fermare a documentarmi / studiare il funzionamento della **KV cache**.

---

## 2. La questione della KV cache

Come detto la lentezza osservata non dipendeva tanto dal throughput di generazione, quanto da un fattore più nascosto: la **KV cache**.

In un modello transformer, ogni layer elabora una sequenza di token tramite il meccanismo di **self-attention**, che si basa su tre matrici di valori numerici:

* **Q** (*Query*),
* **K** (*Key*),
* **V** (*Value*).

Quando il modello genera un token, confronta il vettore `Q` di quel token con tutti i vettori `K` e `V` dei token precedenti per capire su quali “prestare attenzione”.
In altre parole, le matrici **Key** e **Value** rappresentano la memoria a breve termine del modello:
ogni token passato aggiunge una coppia (`K`, `V`) per ciascun layer.

Ricalcolare tutte queste coppie a ogni passo è computazionalmente molto pesante e per evitarlo, i motori di inferenza mantengono in memoria una **cache KV**, cioè una struttura che conserva i vettori `K` e `V` dei token già elaborati.
Quando arriva un nuovo token, il modello valuta se può riusare direttamente questi dati senza ricomputarli da zero.

Chiaramente mantenere la cache richiede molta memoria, ma il risparmio di tempo ogni qual volta è possibile riusare dati dalla cache è immenso.

Per massimizzare l'utilizzo della cache, si utilizzano *KV cache manager* che utilizzano varie tecniche per identificare prompt con una parte di contesto identica o fortemente sovrapposta a quello di richieste precedenti, così da poter riutilizzare le informazioni contenute nella cache. 

LM Studio si basa internalmente sulla libreria llama.cpp che mi risulta disporre di un meccanismo di KV cache, anche se non particolarmente avanzato. Sequenze di prompt profondamente differenti o semplicemente di lunghezza molto differente, invalidano facilmente la cache, portando a ricalcoli per ricostruire la rappresentazione interna del contesto, rieseguendo l'attenzione per tutti i token dei prompt.
Su 64 k token di input, parliamo di diverse decine di secondi di lavoro aggiuntivo sul mio hardware.

Nel mio caso di test si verificava esattamente questo fenomeno, presumibilmente in relazione alla sequenza con cui i vari tipi differenti di prompt vengono generati ed inviati dal coding agent.
Non ho fatto vera e propria analisi del traffico HTTP tra agente e modello, ma è abbastanza evidente che il router inoltra richieste tutt'altro che omogenee: ci sono chiamate che coinvolgono interi file, ragionamenti multi-step, sintesi di piani d’azione — e altre molto più brevi, che servono solo per gestire il flusso, interrogare lo stato, o verificare la disponibilità di tool.
Sono proprio queste micro-richieste, alternate alle operazioni lunghe, che più frequentemente portano all’invalidazione della cache.

Questo spiega perché il tempo percepito “prima del primo token” è così alto.

---

## 3. Seconda configurazione: due istanze del modello

Dopo aver capito che la lentezza non dipendeva solo dal throughput, ma anche dal modo in cui la cache KV veniva continuamente invalidata, la direzione successiva era quasi obbligata: **separare le richieste “piccole” da quelle “grandi”**.

Decido quindi di ispirarmi all’architettura di Claude Code originale così come recepita nella configurazione del router: usare **più modelli (o istanze dello stesso modello)**, ciascuno dedicato a un tipo di richiesta diverso.

---

### Configurazione a due istanze

Nel mio caso, ho mantenuto `gpt-oss-20b` ma l’ho avviato due volte in LM Studio, con due profili distinti e due dimensioni massime di contesto differenti, rispettivamente 16 k e 64 k.
Nel router la configurazione diventava:

```json
{
   ...
  "Providers": [
    {
      "name": "lmstudio-small",
      "models": ["openai/gpt-oss-20b:2"],
      "default_params": { "max_tokens": 1024 }
    },
    {
      "name": "lmstudio-long",
      "models": ["openai/gpt-oss-20b"],
      "default_params": { "max_tokens": 32000 }
    }
  ]
  
  "Router": {
    "default": "lmstudio-small,openai/gpt-oss-20b:2",
    "background": "lmstudio-small,openai/gpt-oss-20b:2",
    "think": "lmstudio-long,openai/gpt-oss-20b",
    "longContext": "lmstudio-long,openai/gpt-oss-20b",
    "longContextThreshold": 3000,
    "webSearch": "lmstudio-small,openai/gpt-oss-20b:2",
    "image": "lmstudio-small,openai/gpt-oss-20b:2"
  }
  ...
}
```

Il router ora aveva due percorsi distinti:

* `lmstudio-small` gestiva le richieste veloci e di servizio (quelle che prima invalidavano più spesso la cache).
* `lmstudio-long` era riservato alle operazioni di reasoning esteso e alle generazioni di codice complesse.

---

### Perché `max_tokens` diversi

Il parametro `max_tokens` definisce quanti token **la risposta del modello** può generare in un’unica inferenza.
Ma questo valore incide direttamente anche su **quanto spazio di cache viene riservato** durante l’elaborazione:
il motore deve infatti allocare la memoria necessaria non solo per il contesto in ingresso, ma anche per i token in uscita *potenziali*.

Un limite alto (come 32 k token) amplia la finestra di generazione ma aumenta la memoria preallocata per la KV cache; un limite basso (come 1024) riduce l’impatto sulla memoria e accelera la latenza di avvio, soprattutto se le richieste sono brevi.

In altre parole:

* **lmstudio-small** poteva permettersi di rispondere in modo rapido perché manteneva una cache ridotta, perfetta per prompt leggeri e brevi risposte.
* **lmstudio-long**, invece, era configurato per reasoning lunghi e poteva lavorare su contesti estesi, ma a costo di tempi di inizializzazione più alti e maggiore uso di memoria.

---

### Risultati empirici

Con questa configurazione la differenza si è vista subito.
Le operazioni leggere — ad esempio richieste di stato o piccole elaborazioni — diventavano quasi istantanee (2-3 secondi), mentre le sessioni reasoning lunghe restavano lente ma più prevedibili, e soprattutto nella maggior parte dei casi **non influenzavano più le richieste successive** (nel senso che non portavano ad invalidazione della cache).

L’agente, però, non riusciva ancora a completare correttamente il primo task del progetto — l’implementazione del *rate limiting* nel servizio — ma almeno ora il comportamento era più coerente e ripetibile, segno che la struttura di base era solida.

{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-model-lying.png" alt="Model fails to update documentation" caption="Il modello fatica a completare il task correttamente, sostenendo di aver aggiornato la documentazione quando in realtà non ci è riuscito." %}

---

## 4. Esperimenti con Qwen 3

Ho provato a sostituire il modello con **Qwen 3**.
Il primo tentativo con `Qwen3-Coder-30B-AB3-Instruct` non ha funzionato come speravo: ottimo sul codice, ma senza capacità reasoning sufficiente per la fase *think*.
Sono quindi passato a [`Qwen3-30B-A3B-Instruct-2507`](https://huggingface.co/Qwen/Qwen3-30B-A3B-Instruct-2507), che migliora il precedente in particolare aggiungengo le capacità di reasoning.

La configurazione era la stessa: due istanze LM Studio, stesso schema di router.
Le risposte migliorano, ma il ciclo completo (feature → test → doc) ancora non si chiudeva correttamente.

{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-qwen-circle.png" alt="Model loop" caption="Il modello va in loop cercando invano di modificare i sorgenti" %}
{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-qwen-config-ctx-full.png" alt="Model ctx full" caption="... di conseguenza, il contesto si riempie, causando ulteriori problemi." %}

---

## 5. La soluzione finale: GPT-OSS-120B

A quel punto ho deciso di passare al livello successivo: **[GPT-OSS-120B](https://huggingface.co/openai/gpt-oss-120b)**, caricato in LM Studio sempre con Flash Attention attiva, tutti i layer in GPU e contesto massimo fissato a 64k.
L’ho assegnato ai ruoli `think` e `longContext` nella configurazione del router, mantenendo `gpt-oss-20b` per le richieste leggere e di background.

```json
{
  ...
  "Providers": [
    {
      "name": "lmstudio-small",
      "models": ["openai/gpt-oss-20b"],
      "default_params": { "max_tokens": 1024 }
    },
    {
      "name": "lmstudio-long",
      "models": ["openai/gpt-oss-120b"],
      "default_params": { "max_tokens": 32000 }
    }
  ]
  
  "Router": {
    "default": "lmstudio-small,openai/gpt-oss-20b",
    "background": "lmstudio-small,openai/gpt-oss-20b",
    "think": "lmstudio-long,openai/gpt-oss-120b",
    "longContext": "lmstudio-long,openai/gpt-oss-120b",
    "longContextThreshold": 3000,
    "webSearch": "lmstudio-small,openai/gpt-oss-20b",
    "image": "lmstudio-small,openai/gpt-oss-20b"
  }
  ...
}
```

Con questa configurazione i tempi di reasoning erano più stabili, il contesto mantenuto correttamente e, finalmente, il ciclo di sviluppo completato in modo coerente.

{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-gpt-oss-120b-1-analysis.png" alt="gpt-oss 120B - analysis" caption="Analizziamo il progetto con il modello da 120B" %}
{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-gpt-oss-120b-2-fix.png" alt="gpt-oss 120B - 2" caption="Inizio implementazione rate limiting feature..." %}
{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-gpt-oss-120b-3-fix.png" alt="gpt-oss 120B - 3" caption="... completiamo l'implementazione." %}
{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-gpt-oss-120b-4-test.png" alt="gpt-oss 120B - 4" caption="Preparazione test della nuova feature" %}
{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-gpt-oss-120b-5-test.png" alt="gpt-oss 120B - 5" caption="Test eseguiti correttamente." %}

Il tempo medio per richiesta utente si è assestato intorno ai **30 secondi**, con variazioni in base al tipo di operazione e al numero di file coinvolti.
Un risultato tutt’altro che “snello”, ma perfettamente accettabile considerando che tutto girava su una singola macchina.

{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-gpt-oss-120b-config-memory.png" alt="gpt-oss 20B + 120B - config" caption="Configurazione definitiva e memoria utilizzata." %}
{% include figure image_path="/assets/images/coding-agent-locale/coding-agent-locale-gpt-oss-120b-running-cache-reuse.png" alt="gpt-oss 20B + 120B - running" caption="Modelli da 120B e 20B in esecuzione in contemporanea e cache correttamente utilizzata." %}

---

## 6. Considerazioni

Dopo vari giorni di prove, questi sono alcuni dei punti che mi porto a casa:

* **Agentic coding è un carico completamente diverso dalla chat.**
Un agente deve pianificare, leggere e aggiornare continuamente il contesto, quindi non basta che il modello "si carichi e giri”.

* **La memoria è spesso il collo di bottiglia.**
  Ogni incremento nel contesto massimo si traduce in gigabyte di memoria aggiuntiva.
  La configurazione a due modelli descritta qui sopra portava la mia macchina ad utilizzare complessivamente attorno a 90 GB di memoria.

* **Separare i modelli serve.**
  Quantomeno per come pare essere il funzionamento di Claude Code, e in assenza di un KV cache manager evoluto, un setup multi-modello (piccolo + grande) aiuta a non invalidare continuamente la cache e migliora la reattività percepita.

* **La velocità di generazione conta meno della latenza iniziale.**
  In locale, l’attesa per il primo token è spesso il fattore dominante.

---

## 7. Conclusione

Mi sento di dire che portare un *coding agent* in locale (per singolo utente) oggi è sostanzialmente possibile, ma richiede hardware generoso, pazienza e un po’ di curiosità tecnica.
Il risultato finale non sostituisce l’esperienza fluida di Claude Code utilizzato attraverso le API, ma permette di capire meglio **cosa succede sotto il cofano**:
quali risorse servono, come incide il contesto, cosa fa la cache, etc.
In termini di qualità nel coding, quanto proposto dal `gpt-oss-120b` è a prima vista buono, ma mi aspetto che al crescere della complessità del progetto, il divario con i risultati ottenibili utilizzando le API a modelli remoti si percepisca in modo evidente.

Infine, chiaramente la scelta di usare Claude Code Router nasceva dall'idea di provare uno scenario il più simile a quello testato nel post del mese scorso; per quanto l'idea di base del tool sia interessante, non è necessariamente la migliore per il caso d'uso descritto in questo post. Nelle prossime settimana non escludo di provare altri agenti, magari pensati proprio per l'esecuzione con modelli locali.
