---
title: "Coding agent: modelli locali vs API"
date: 2025-11-03
excerpt: "Proviamo Cline e Kilo Code con modelli locali e remoti (via API) e analizziamo i risultati per comprendere cosa cambia davvero."
tags: [AI locale, LLM, Cline, Kilo Code, aiopenweight, Strix Halo, generativeai]
author: alessio
classes: wide
header:
  overlay_image: /assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-C.jpg
  overlay_filter: 0.4
---

A inizio Novembre mi sono cimentato in alcuni test di sviluppo software attraverso Claude Code integrato (via Claude Code Router) con modelli openweight GPT OSS eseguiti in locale.

Nelle ultime settimane, invece, ho voluto sperimentare due coding agent specificamente concepiti per servirsi di svariati modelli (openweight e commerciali) in locale o in remoto. Nello specifico, ho provato [Kilo Code](https://kilo.ai/) e [Cline](https://cline.bot/); entrambi sono disponibili come estensioni di molti dei principali IDE, tra i quali ho scelto [VS Code](https://code.visualstudio.com/).

## Lo scenario di test
Come scenario di test, di nuovo ho scelto di "proseguire lo sviluppo" del piccolo progetto opensource ([onnx-model-generator-docker](https://github.com/asoldano/onnx-model-generator-docker)) che già avevo usato in precedenza: in sintesi, ricordo che si tratta di un servizio REST Dockerizzato che converte modelli HuggingFace in formato Open Neural Network Exchange (ONNX) utilizzando la libreria _onnxruntime-genai_: riceve una richiesta HTTP e restituisce un modello ottimizzato pronto da scaricare.

In precedenza avevo chiesto a Claude di elencare le possibili migliorie, in particolare in ambito di sicurezza, e di salvarne la lista insieme alla documentazione di progetto; ho dunque utilizzato il seguente semplice prompt per dare il via alle operazioni di Cline e Kilo Code in tutte le prove qui descritte:
 
``
Hi, please have a look at the project, including the .md files and figure out what the next improvement task to work on would be, then prepare for completing it (including  testing and updating the existing .md documents)
``

Il primo task su cui lavorare, dato lo [stato corrente](https://github.com/asoldano/onnx-model-generator-docker/tree/05996137294a7802914caacd0130444242080bc8) del progetto, era quello di aggiungere il _rate limiting_, ossia permettere di controllare il numero massimo di richieste al minuto verso il servizio Docker da un dato indirizzo IP.


## I modelli utilizzati
Per quanto concerne gli LLM utilizzati con i due coding agents, ho deciso di testare principalmente i [modelli GLM di Z.ai](https://z.ai/model-api). Tra questi, [**GLM 4.6**](https://docs.z.ai/guides/llm/glm-4.6#introducting-glm-4-6) si posiziona al livello di alcuni degli attuali modelli SOTA ed è disponibile per utilizzo remoto via API a prezzi decisamente ridotti rispetto alla concorrenza (il che presumo ne faccia o farà una scelta popolare). Inoltre, il modello è openweight, [disponibile su huggingface.co](https://huggingface.co/zai-org/GLM-4.6) e dunque utilizzabile anche in locale... almeno in linea teorica.

Nella pratica, trattandosi di un modello da **357 miliardi di parametri**, l'inferenza su qualunque hardware consumer attuale è sostanzialmente preclusa a meno di ricorrere a quantizzazioni estremamente aggressive.

Per le prove in locale sul mio AMD Ryzen AI Max+ 395 (Strix Halo) con 128 GB di RAM condivisa ho dunque scelto di provare **GLM 4.5-Air**, ossia la versione snellita (**106 miliardi di parametri** in totale, 12 attivi - _Mixture of Experts_) di GLM 4.5, il modello precedente a GLM 4.6. GLM 4.5 è pubblicizzato come qualitativamente ancora comparabile con Claude 4 Sonnet, mentre 4.5-Air è dato come solo leggermente inferiore a 4.5 in termini di qualità della generazione. Per ridurre al minimo l'impatto della quantizzazione, ho scelto di utilizzare la [versione GGUF Q5_K_M diffusa su huggingface.co da Unsloth AI](https://huggingface.co/unsloth/GLM-4.5-Air-GGUF?show_file_info=Q5_K_M%2FGLM-4.5-Air-Q5_K_M-00001-of-00002.gguf). Ho effettuato il deploy del modello su [LMStudio](https://lmstudio.ai/), eseguendo tutti i layer in GPU e abilitando la Flash Attention.

Infine, ho anche rieseguito qualche prova con Cline abbinato allo stesso [GPT-OSS-120B](https://huggingface.co/openai/gpt-oss-120b) usato in precedenza, per avere un riferimento.


## Kilo Code

### Configurazione
Dopo aver completato l'installazione in VS Code, Kilo Code si configura per connettersi a differenti tipi di modelli LLM utilizzando la scheda _Providers_ all'interno della sezione _Settings_.
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-conf.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-conf.jpg" alt="Configurazione Kilo Code" style="margin-bottom:20px;" />
</a>
Nel caso dei modelli remoti, ho selezionato _Z AI_ come API provider (_International_ entrypoint) e copiato la mia API key generata su sito di Z.ai dopo aver sottoscritto il piano _Lite_ (9 USD a trimestre); ho abilitato il _reasoning_ e lasciato invariate tutte le altre impostazioni.

Per utilizzare il modello GLM 4.5-Air in locale, invece, prima di tutto ho avviato il server LMStudio ed eseguito il deploy del modello utilizzando una finestra di contesto di **65k** token, così da avere un buon compromesso tra velocità di inferenza / memoria utilizzata e lunghezza della "conversazione" tra agente e modello. Poi nelle impostazioni di Kilo Code ho selezionato _LM Studio_ della lista di API Provider supportati e scelto _glm-4.5-air_ tra le opzioni presentate automaticamente (ottenute da Kilo Code interrogando il server LM Studio attraverso Open API). Anche qui non ho apportato ulteriori cambiamenti alle impostazioni avanzate.

### Modalità di funzionamento
Kilo Code può adattare il suo comportamento a seconda della [modalità (mode) di funzionamento](https://kilo.ai/docs/basic-usage/using-modes) selezionata. Le modalità disponibili di default (è possibile infatti impostarne di personalizzate) tra cui scegliere sono ``Code``, ``Ask``, ``Architect``, ``Debug`` e ``Orchestrator``; ognuna di esse si caratterizza per differenti "personalità", ottimizzazioni e accesso (nonché permesso di esecuzione) a tool: lettura/scrittura file, MCP, browser, comandi shell, ... L'idea è che si scelga la modalità di esecuzione più adatta al compito da compiere.

Per semplicità, ho selezionato ``Architect`` per la prima fase di esecuzione (il prompt parla di "comprendere quale sia il prossimo task..." e di "prepararsi a completarlo"), per poi passare in modalità ``Code`` per la successiva fase implementativa.

### Prove effettuate

Come anticipato sopra, ho sperimentato tre differenti scenari di utilizzo, con tre differenti modelli / API Provider: GLM 4.6 in remoto, GLM 4.5-Air in remoto, GLM 4.5-Air in locale. Qui sotto riporto una selezione di screenshot catturati durante le prove.

**GLM 4.6 in remoto**

<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-A.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-A.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-B.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-B.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-C.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-C.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-D.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-D.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-E.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-E.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-F.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-F.jpg" style="height:110px;margin:6px;">
</a>

**GLM 4.5-Air in remoto**

<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-A.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-A.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-B.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-B.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-C.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-C.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-D.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-D.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-E.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-E.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-F.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm45air-F.jpg" style="height:110px;margin:6px;">
</a>

**GLM 4.5-Air in locale**

<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-A.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-A.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-B.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-B.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-C.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-C.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-D.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-D.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-E.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-E.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-F.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-lms-glm45air-F.jpg" style="height:110px;margin:6px;">
</a>

In tutti gli scenari il compito richiesto è stato completato con successo: l'agente ha compreso quale fosse il prossimo task a cui lavorare ed è stato in grado di completarlo (nel caso di GLM 4.5-Air locale, a dire il vero, ha preparato una lista di task a cui lavorare, il primo dei quali quello che ci si attendeva).

In puro stile _"vibe coding"_, ho lasciato lavorare l'agente in autonomia, rispondendo a domande chiuse (scelta multipla) e confermando manualmente le sole esecuzioni dei tool (per le testsuite) e il passaggio alla fase di coding. Le analisi condotte nella prima fase hanno evidenziato differenti livelli di approfondimento e chiarezza nello spiegare il piano; in accoppiata con GLM 4.6, Kilo ha fatto un ottimo lavoro generando, tra le altre cose, una domanda con 4 possibili risposte pre-confezionate (per verificare le intenzioni dell'utente) e un diagramma di flusso della logica da implementare.

<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-domande.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-domande.jpg" alt="Domande a scelta multipla" style="margin-bottom:20px;" />
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-domande.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/kilo-glm46-diagramma-flusso.jpg" alt="Domande a scelta multipla" style="margin-bottom:20px;" />
</a>

In tutti i casi il sistema è stato in grado di **generare test** per la nuova funzionalità (risolvendo fallimenti quando si sono presentati) ed **aggiornare la documentazione**.

Per quanto concerne i tempi di esecuzione, non è possibile fare un vero confronto, considerato il non determinismo che porta a percorsi di risoluzione del problema ed esecuzione completamente differenti, tuttavia la versione in locale è risultata in generale leggermente più lenta, ma la differenza è stata meno significativa di quanto mi aspettassi (Kilo spende diverso tempo anche "lato client"). Va comunque considerata anche la **differente dimensione del contesto**, l'utilizzazione del quale è chiaramente visibile negli screenshot.


## Cline

### Configurazione
Anche nel caso di Cline ho installato l'estensione VS Code. Una volta completata la procedura, di nuovo come per Kilo Code, il tab _Settings_ di Cline consente di specificare l'_API Provider_ (_Z.ai_ e _LM Studio_, nel mio caso) e di fornire rispettivamente la chiave o l'URL del server e il nome del modello da utilizzare. Non dovendo utilizzare finestre di contesto molto piccole (inferiori a 8k), non ho abilitato l'utilizzo di prompt compatti.

<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-conf.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-conf.jpg" alt="Configurazione Cline" style="margin-bottom:20px;" />
</a> 

### Modalità di funzionamento
Se Kilo Code dispone di diversi _modes_, Cline invece distingue tra ``Plan mode`` e ``Act mode``. È possibile configurare Cline affinché utilizzi modelli differenti per le due modalità, selezionando ad esempio un modello _reasoning_ in ``Plan`` e uno non _reasoning_ in ``Act``.

Considerato il prompt scelto per questi esperimenti, sono sempre partito con la modalità ``Plan``, per poi autorizzare Cline a passare in modalità ``Act`` una volta preparato il piano d'intervento.

### Prove effettuate
Con questo agente ho testato nuovamente GLM 4.6 remoto, mentre in locale ho provato ad utilizzare, sempre via LM Studio, _GPT-OSS-120B_ che discrete soddisfazioni aveva dato negli [esperimenti con Claude Code Router](https://aladinodigitale.it/2025/11/02/coding-agent-locale.html) (utilizzato però in parallelo ad un modello più piccolo, _GPT-OSS-20B_).

Ho utilizzato Cline in maniera simile a Kilo Code, sostanzialmente autorizzando sempre le richieste di utilizzo dei tool (nella configurazione di default, Cline lascia maggiore controllo all'utente rispetto a Kilo Code, che applica le modifiche ai file senza chiedere conferma) e lasciando che cercasse autonomamente di superare i problemi incontrati, risolvendo bachi, installando dipendenze, etc.

Qui sotto alcune immagini catturate durante l'esecuzione, come nel caso precedente. Da notare che, nella configurazione con GPT-OSS-120B, due tentativi di eseguire nuovamente il compito richiesto nel prompt sono falliti, con l'agente che non è riuscito a produrre un'implementazione funzionante che passasse i test.

**GLM 4.6 in remoto**

<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-A.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-A.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-B.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-B.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-C.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-C.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-D.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-D.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-E.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-E.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-F.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-glm46-F.jpg" style="height:110px;margin:6px;">
</a>

**GPT-OSS-120B in locale**

<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-A.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-A.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-B.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-B.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-C.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-C.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-D.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-D.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-E.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-E.jpg" style="height:110px;margin:6px;">
</a>
<a href="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-F.jpg" target="_blank">
  <img src="/assets/images/coding-agent-modelli-locali-vs-api/cline-lms-gpt-F.jpg" style="height:110px;margin:6px;">
</a>

## Analisi risultati

Sebbene sia stato possibile completare il task richiesto in tutti gli scenari, **la qualità delle soluzioni ottenute non è necessariamente la stessa**. Invece di effettuare il merge di una delle implementazioni sul main del progetto, ho creato un branch differente per ognuno degli scenari provati:
* Kilo Code + GLM 4.6 remoto: https://github.com/asoldano/onnx-model-generator-docker/tree/kilo-glm-4.6
* Kilo Code + GLM 4.5-Air remoto: https://github.com/asoldano/onnx-model-generator-docker/tree/kilo-glm-4.5-air
* Kilo Code + GLM 4.5-Air locale:https://github.com/asoldano/onnx-model-generator-docker/tree/kilo-lms-glm-4.5-air-65k
* Cline + GLM 4.6 remoto: https://github.com/asoldano/onnx-model-generator-docker/tree/cline-glm-4.6-200k
* Cline + GPT-OSS 120B locale: https://github.com/asoldano/onnx-model-generator-docker/tree/cline-lms-gtp-oss-120b-64k 

Per una valutazione imparziale e competente dei risultati, ho poi generato le patch corrispondenti alle modifiche su ognuno dei branch a partire dal branch point e le ho sottomesse a *Google Gemini 3*.

``
Ciao! ho bisogno che tu mi aiuti a valutare la bontà di 5 differenti implementazioni della stessa feature sulla stessa codebase. Il punto di partenza è https://github.com/asoldano/onnx-model-generator-docker , ti allego le 3 patch delle tre implementazioni alternative, la feature da implementare era rate limiting, ma ti invito a leggere il sorgente del progetto dal main , guardare i file .md e capire cosa è stato scelto di implementare (ho usato tool AI per farlo, il prompt era: "hi , please have a look at the project, incuding the .md files and figure out what the next improvement task to work on would be, then prepare for completing it (including testing and updating the existing .md documents)" . Per favore, analizza le tre proposte e dimmi come si posizionano come qualità.
``

Il risultato dell'analisi effettuata per me dall'intelligenza artificiale stessa evidenzia **punti di forza e debolezze** delle patch. Chi fosse curioso può provare ad interrogare Gemini (o magari Claude?) e leggere la risposta; qui invece ritengo sia interessante focalizzarsi sulle **aree di debolezza** individuate (a prescindere da quale configurazione abbia portato ad uno dei comportamenti sub-ottimi), così da _capire dove sia il vero valore aggiunto di utilizzare un modello "migliore"_:
* **Hardcoding di parametri** con valori arbitrari
* **Stile di testing incoerente** (stampe in standard output, mancanza di assertion / exit code appropriati)
* **Caos nella gestione delle dipendenze** (downgrade, aggiunte e rimozioni inappropriate di librerie)
* Scelte di **design non ottimali** (nel nostro caso, un rate limit unico per tutti gli endpoint)
* **Race conditions / implementazione non thread-safe**
* **NIY syndrome** (implementazione custom dove esistono librerie stabili e testate che offrono la stessa funzionalità)

Per quanto concerne invece la scelta dell'agente o del _modello "migliore"_, Gemini ha chiaramente fornito una sua classifica delle 5 soluzioni presentate, ottenuta dando un peso ai vari pro e contro evidenziati; sul podio abbiamo:
1. Kilo Code + GLM 4.6 remoto
2. Cline + GLM 4.6 remoto
3. Kilo Code + GLM 4.5-Air remoto

In ottica di merge sul main del progetto, probabilmente per mantenere una certa neutralità, Gemini consiglia di derivare una _soluzione "perfetta"_ partendo da quella di Cline + GLM 4.6 remoto e aggiungendo alcune specifiche idee riscontrate unicamente nelle altre soluzioni (incluse le due ottenute con modello locale).

## Conclusioni
Prima di saltare alle conclusioni, alcune considerazioni:
* l'esperimento si basa chiaramente su un'osservazione estremamente limitata e va considerata la natura indeterministica / probabilistica dei modelli... quindi non ci sbilanciamo più tanto ;-)
* ci sono sicuramente margini di miglioramento / tuning a livello della configurazione degli agenti Cline e Kilo Code.

Detto questo, dopo le prove qui descritte ecco i punti che ci portiamo a casa:
* Il ranking delle soluzioni riportato dal Gemini 3 (probabilmente la più avanzata AI disponibile al momento) rispecchia quello che ci saremmo aspettati considerando le specifiche e i benchmark dei modelli: **GLM 4.6 meglio di GLM 4.5-Air**, **GLM 4.5-Air in remoto via API meglio della versione quantizzata in locale**.
* È possibile portare a termine egregiamente un task di sviluppo come quello qui presentato in tutte le modalità presentate; abbiamo però un'idea della **direzione in cui guardare per apprezzare le differenze di qualità tra soluzioni prodotte con modelli più o meno performanti**; questo ci _aiuta anche in generale nell'ottimizzare la review umana di codice generato dall'AI_.
