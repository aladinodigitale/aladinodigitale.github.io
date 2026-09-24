---
title: "Quando l'open source sblocca il silicio: il caso AMD e l'inferenza AI locale"
date: 2026-09-24
excerpt: "Il vantaggio di NVIDIA nell'AI non sta solo nel silicio, ma nello software che lo circonda. Attorno ad AMD Strix Halo sta però emergendo un ecosistema open source che sta imparando a sfruttare l'hardware meglio e più in fretta di quanto il vendor da solo avrebbe potuto: formati di quantizzazione dedicati, runtime che sfruttano istruzioni specifiche della GPU e ottimizzazioni nate dalla community."
tags: [AI locale, open source, AMD, Strix Halo, ROCm, llama.cpp, quantizzazione, LLM, inferenza]
author: alessio
classes: wide
header:
  overlay_image: /assets/images/open-source-amd-strix-halo/overlay.jpg
  overlay_filter: 0.5
---

Quando si parla del vantaggio di NVIDIA nell'intelligenza artificiale, si tende naturalmente a pensare alle GPU. Più potenza di calcolo, più memoria, Tensor Core sempre più sofisticati.

Ma una parte enorme di quel vantaggio non è nel silicio.

È nel software.

CUDA esiste dal 2006. In quasi vent'anni NVIDIA ha costruito intorno alle proprie GPU un ecosistema verticale composto da compiler, runtime, librerie matematiche, kernel ottimizzati, profiler, framework e strumenti di sviluppo. Quando un nuovo modello arriva sul mercato, nella maggior parte dei casi buona parte del lavoro necessario per farlo girare bene su NVIDIA è già stata fatta.

AMD parte da una posizione diversa. ROCm è sviluppato direttamente da AMD ed è in larga parte open source, ma è un ecosistema più giovane e ancora meno uniforme. Su hardware consumer recente non è raro incontrare feature non ancora ottimizzate, percorsi software alternativi, differenze tra [HIP](https://rocmdocs.amd.com/projects/HIP/) e [Vulkan](https://www.amd.com/en/products/graphics/ecosystems/vulkan.html), regressioni o semplicemente workload che nessuno aveva ancora studiato in dettaglio.

A prima vista può sembrare solamente uno svantaggio.

Negli ultimi mesi, però, osservando quello che sta succedendo intorno a hardware come [Strix Halo](https://www.amd.com/en/products/processors/laptop/ryzen/ai-300-series/amd-ryzen-ai-max-plus-395.html), ho iniziato a vederci anche qualcosa di molto più interessante.

Perché quando lo stack è aperto, un collo di bottiglia non deve necessariamente aspettare che il vendor decida di risolverlo.

Può farlo qualcun altro.

E qualcun altro ancora può prendere quella soluzione, modificarla, misurarla, combinarla con un'altra idea e infine tentare di riportarne la parte generalizzabile upstream.

Quello che sta emergendo non è semplicemente "una community che rende AMD più veloce".

È qualcosa che assomiglia sempre di più a un **co-design distribuito tra modelli, rappresentazioni numeriche, algoritmi di inferenza, runtime e hardware**.

---

## Il primo cambio di prospettiva: quantizzare non significa solo occupare meno memoria

Per anni abbiamo raccontato la quantizzazione quasi esclusivamente come un compromesso.

Un modello a 4 bit occupa meno memoria di uno a 8 o 16 bit, al prezzo di una certa perdita di precisione.

È vero, naturalmente. Ma su una macchina come Strix Halo manca un pezzo importante della storia.

Durante il decode di un LLM, molto spesso il problema non è la quantità di operazioni che la GPU può teoricamente eseguire. Il problema è la velocità con cui i pesi possono essere letti dalla memoria.

Se per generare ogni token devo attraversare decine di gigabyte di pesi, leggere meno bit significa anche spostare meno dati.

La quantizzazione diventa quindi una tecnica per sfruttare meglio la memory bandwidth.

È in questo contesto che lavori come **[ROCmFPX](https://github.com/charlie12345/ROCmFPX) di [Carlo Pasquale](https://github.com/charlie12345)** diventano particolarmente interessanti.

ROCmFP4 e le successive varianti non sono semplicemente nuovi nomi per una Q4 tradizionale. Sono formati e layout pensati esplicitamente per l'esecuzione su hardware AMD, con compromessi differenti tra dimensione, precisione e velocità. Nel repository compaiono oggi diversi percorsi specifici per Strix Halo, sia HIP sia Vulkan.

Ma il passaggio concettuale successivo è ancora più importante.

Forse la domanda sbagliata è:

"Questo modello è Q4 o Q8?"

La domanda più interessante diventa:

**quali pesi conviene quantizzare, a quale precisione, e per quale motivo?**

Qui è utile anche il lavoro di **Salvatore Sanfilippo, [antirez](https://github.com/antirez), con [DwarfStar/DS4](https://github.com/antirez/ds4)**.

Nelle ricette per i grandi modelli MoE, per esempio, non tutti i tensori vengono trattati allo stesso modo. I routed experts possono essere compressi molto aggressivamente, mentre altre componenti — proiezioni, shared experts, output — vengono mantenute a precisione superiore. Anche l'importance matrix viene usata per guidare la quantizzazione.

Non è più semplicemente "comprimere un modello".

È una forma di **allocazione del budget di precisione**.

Spendere più bit dove servono davvero e meno dove il modello può permetterselo.

---

## Quando la quantizzazione entra nel runtime

Il lavoro di **[Ciru](https://huggingface.co/jcbtc/activity/community)** parte da ROCmFPX e spinge questa idea ancora oltre.

[DualView](https://llm.ciru.ai/dualview#explainer), per esempio, parte da una rappresentazione Q7 dei pesi, ma permette alla GPU di vederli in modi diversi a seconda della fase dell'inferenza.

Durante il decode usa la rappresentazione compatta, riducendo i byte che devono essere letti per ciascun token.

Durante il prefill espone invece gli stessi valori sotto forma di una compute view Q8, adatta ai percorsi INT8 nativi dell'hardware.

Non viene fatta una seconda quantizzazione: cambia la rappresentazione utilizzata per il calcolo.

È un dettaglio tecnico che nasconde un'idea molto più generale:

**non esiste necessariamente una rappresentazione numerica migliore in assoluto. Può esistere una rappresentazione migliore per ciascuna fase dell'inferenza.**

ActiveFPX e PromptForge proseguono nella stessa direzione, arrivando a costruire percorsi di esecuzione specifici per le forme delle matrici effettivamente prodotte da modelli come Qwen 3.8.

A quel punto diventa difficile tracciare una linea netta tra "formato del modello" e "runtime".

La quantizzazione è ormai parte del motore di esecuzione.

---

## Quando un modello sblocca un'istruzione della GPU

Un altro esempio che trovo particolarmente elegante riguarda MTP, Multi-Token Prediction.

Nel normale decode autoregressivo un LLM genera essenzialmente un token alla volta.

Questo limita il parallelismo disponibile in alcune operazioni.

Ma con MTP il modello può proporre più token futuri, che vengono poi verificati insieme dal modello principale.

Questa verifica batched cambia la forma del problema.

Ed è qui che entra in gioco una caratteristica molto specifica di Strix Halo: le istruzioni matrix IU4 disponibili su gfx1151 (Strix Halo).

Nel lavoro ROCmFPX, un percorso sperimentale W4A4 collega proprio questi due mondi: pesi a quattro bit, attivazioni a quattro bit e istruzioni matrix IU4.

La cosa interessante non è tanto il risultato numerico del benchmark. È la catena causale:

**il modello introduce MTP → MTP crea una verifica batched → la verifica batched rende utile una determinata operazione matrix → il runtime può finalmente sfruttare un'istruzione hardware specifica.**

A questo punto non stiamo più semplicemente ottimizzando un kernel.

Stiamo collegando una caratteristica dell'architettura del modello a una caratteristica della microarchitettura della GPU.

Ed è proprio questo uno degli aspetti più interessanti dell'open source: chi sviluppa il runtime può osservare entrambe le estremità del problema.

---

## Poi qualcuno apre il profiler

Non tutte le ottimizzazioni nascono da un nuovo formato numerico.

[llama.cpp](https://github.com/ggml-org/llama.cpp) ha oltre 23 mila fork; tra questi, diversi sono oggetto di lavoro di ricerca per ottimizzazioni specifiche per determinati scenari, hardware, etc. Tra queste, interessante è il lavoro di **[Nathan Wilson](https://github.com/Nathanw1014)**, il cui fork per Strix Halo rappresenta bene l'altra faccia del processo di ottimizzazione: profiling sistematico, identificazione dei colli di bottiglia, patch piccole e verificabili.

Un esempio riguarda Flash Attention e la KV cache quantizzata. In alcuni percorsi il runtime finiva per dequantizzare gli stessi dati più volte. Modificando il flusso in modo da effettuare la dequantizzazione una sola volta e riutilizzarne il risultato, si elimina lavoro ridondante.

Lo stesso repository contiene interventi su MoE prefill, percorsi MMID, layout dei dati e kernel HIP/Vulkan. Ma forse la parte più importante è un'altra: Nathan separa chiaramente le patch candidate all'upstream da quelle che rimangono specifiche per Strix Halo.

Alcuni esperimenti producono miglioramenti importanti, altri quasi nulla, altri ancora peggiorano le prestazioni.

Ed è perfettamente normale.

Open source non significa automaticamente software migliore.

Significa che l'esperimento è visibile, riproducibile e falsificabile.

Qualcun altro può ripeterlo.

Può trovare un workload nel quale non funziona, può capire perché, può proporre una soluzione migliore.

---

## E se il problema non fosse la GPU?

Con modelli sempre più complessi, anche concentrarsi esclusivamente sulla velocità dei kernel GPU comincia a essere limitante. [EngramHalo](https://github.com/Aristo94/EngramHalo.cpp), il fork di [Aristo94](https://github.com/Aristo94) dedicato a Qwen 3.8 Flash-Next, è un buon esempio perché prova a sfruttare alcune caratteristiche particolari dell'architettura del modello invece di trattarlo come un Transformer tradizionale.

Una di queste è Qwen Sparse Attention (QSA). Con la normale attention, all'aumentare del contesto cresce anche la quantità di informazioni passate che bisogna considerare. QSA parte invece dall'idea che, per il token corrente, gran parte di un contesto molto lungo non sia necessariamente rilevante: un piccolo indexer individua quindi i blocchi del passato che vale la pena recuperare, e l'attention può concentrarsi principalmente su quelli.

Ma avere un algoritmo concettualmente sparse non significa necessariamente eseguirlo in modo davvero sparse. Se il runtime individua i token rilevanti ma continua comunque a leggere una grande porzione della KV cache per poi escludere ciò che non serve tramite una maschera, dal punto di vista logico sta facendo sparse attention, ma dal punto di vista della memoria continua a muovere molti dati inutilmente.

Uno degli interventi interessanti di EngramHalo è proprio qui: usa gli indici prodotti da QSA per effettuare un vero sparse gather, raccogliendo prima soltanto le righe K/V selezionate in un buffer compatto e facendo lavorare l'attention direttamente su quello. In termini semplici, invece di leggere tutto e poi scartare ciò che non serve, cerca di leggere direttamente solo ciò che servirà.

Qwen 3.8 Flash-Next introduce poi un'altra struttura particolare: la grande Engram table, una memoria associativa basata su n-gram. Brevi sequenze di token vengono usate per indirizzare direttamente alcune righe di una gigantesca tabella di embedding. La tabella può quindi contenere decine di miliardi di parametri senza che sia necessario attraversarli tutti a ogni token: per ciascuna inferenza ne viene consultata soltanto una piccolissima parte.

Questa caratteristica rende la Engram table molto diversa dai normali pesi di un layer e apre una possibilità interessante: non è indispensabile mantenerla interamente nella memoria più veloce. Può essere lasciata su storage e consultata on demand. EngramHalo sfrutta esplicitamente questa possibilità nel proprio percorso ottimizzato per Strix Halo, combinandola con QSA sparse gather, MTP e altre modifiche al runtime.

Non è un'idea isolata, in realtà. Antirez, in DS4, adotta una strategia molto simile: per Qwen 3.8 Flash-Next i 95,37 GiB di n-gram BF16 rimangono direttamente sul disco e il runtime legge dal GGUF soltanto le righe selezionate, senza caricare o mappare l'intera tabella in memoria. Anche qui il presupposto è lo stesso: se una struttura è enorme ma viene consultata in modo fortemente sparso, tenerla sempre residente è uno spreco.

È qui che il caso EngramHalo, e più in generale il confronto con DS4, diventa particolarmente interessante per il discorso più ampio. L'ottimizzazione non consiste semplicemente nel far eseguire alla GPU la stessa operazione un po' più velocemente. Consiste nel guardare alla struttura del modello e chiedersi quale sia il modo più naturale di mapparla sull'intero sistema.

Una macchina basata su Strix Halo non è soltanto una GPU. È CPU, GPU, cache, memoria unificata e tipicamente NVMe, ciascuna con capacità, latenza e banda differenti. E anche il modello non è più un blocco uniforme di pesi: alcune strutture devono essere attraversate continuamente, altre richiedono accessi sparsi, altre ancora sono enormi ma vengono consultate solo in pochi punti.

Il compito del runtime diventa quindi non soltanto eseguire operazioni matematiche, ma decidere quali dati devono trovarsi dove, quali devono essere mossi e quali possono invece rimanere dove sono finché non servono.

Ed è proprio questo che rende implementazioni come EngramHalo un esempio così interessante del ruolo dell'open source: l'architettura del modello è pubblica, i runtime sono modificabili e il comportamento dell'hardware può essere osservato e misurato. Questo permette a sviluppatori indipendenti di arrivare, anche per strade diverse, a soluzioni simili: sparse attention che sia davvero sparse anche dal punto di vista della memoria, strutture enormi tenute su storage e recuperate on demand, runtime che ragionano sempre meno come semplici esecutori di kernel e sempre più come orchestratori dell'intero sistema.

---

## Quando un'idea esce da una black box

Un passaggio interessante in questa storia è quello che collega [Halogen](https://github.com/peonist-ai/halogen-flash-server), [pwilkin](https://github.com/pwilkin) e Ciru. Halogen, sviluppato da Peonist.ai, è un engine specifico per Qwen 3.8 Flash Next e AMD Strix Halo; mostra quanto si possa spingere lontano l'ottimizzazione quando si costruisce un runtime molto specializzato per una combinazione precisa di modello e hardware, ma il core del motore è proprietario. Il contributo di pwilkin è stato diverso: ha riprodotto in un'implementazione aperta basata su llama.cpp alcune delle idee che permettevano a Halogen di ottenere un prefill particolarmente veloce su Qwen 3.8 Flash-Next, rendendo disponibili kernel, configurazioni e percorso di esecuzione in una forma che altri potessero studiare e modificare.

È proprio da qui che il caso diventa interessante per il tema dell'open source. Una tecnica che inizialmente era osservabile soprattutto attraverso il comportamento di un runtime chiuso diventa improvvisamente codice ispezionabile e riutilizzabile. Ciru ha potuto prendere quel lavoro, adattarlo ai propri formati di peso e al proprio runtime e integrarlo con altre ottimizzazioni. Il valore non sta quindi soltanto nella velocità ottenuta, ma nel passaggio di stato dell'idea: da soluzione incorporata in un prodotto specifico a building block che può essere verificato, modificato e combinato con contributi indipendenti.

È un esempio quasi didattico di come può funzionare la circolazione dell'innovazione in un ecosistema aperto: un'idea emerge in un'implementazione specializzata, qualcuno la ricostruisce in forma open, altri la incorporano e la trasformano ulteriormente. Non è necessario che tutti i pezzi della catena siano open perché questo accada, ma è nel momento in cui l'implementazione diventa aperta che il ritmo di riuso e contaminazione può accelerare davvero.

---

## Dal laboratorio all'ecosistema

C'è però un problema con tutta questa innovazione.

A un certo punto hai:

- un branch di llama.cpp;
- un kernel sperimentale;
- una quantizzazione custom;
- un container diverso;
- una determinata versione di ROCm;
- tre environment variable;
- e un README di quattro pagine.

Tecnicamente funziona.

Praticamente quasi nessuno lo userà.

Qui entra in gioco un contributo meno spettacolare dal punto di vista algoritmico, ma fondamentale per la maturazione dell'ecosistema: quello di **[Donato Capitella kyuz0](https://github.com/kyuz0)**.

[Le AMD Strix Halo Toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes) trasformano molti di questi esperimenti in ambienti riproducibili. Oggi raccolgono stack ROCm e Vulkan, configurazioni per Qwen Flash-Next, EngramHalo e anche l'integrazione con [strix-llama di halo-box](https://github.com/halo-box/strix-llama.cpp).

È facile sottovalutare questo passaggio.

Ma un'ottimizzazione non diventa realmente parte dell'ecosistema quando qualcuno scrive la patch.

Lo diventa quando qualcun altro riesce a riprodurla.

---

## Il problema successivo: troppe buone idee

Arrivati qui emerge un paradosso.

La libertà di sperimentare produce tanti fork; se ciascuna innovazione rimane nel proprio fork, l'ecosistema rischia di frammentarsi.

È qui che progetti come **halo-box/strix-llama.cpp** diventano interessanti.

Il progetto mantiene volutamente una distinzione tra ciò che può essere generalizzato e ciò che rimane specifico di gfx1151. La base rimane vicina a llama.cpp upstream, mentre le ottimizzazioni più aggressive per Strix Halo possono essere integrate e sperimentate separatamente.

Forse è proprio questo il ciclo naturale di maturazione:

**esperimento → fork → benchmark → confronto → integrazione → upstream.**

Non tutte le idee arriveranno alla fine del percorso.

E non devono farlo.

Le ottimizzazioni strettamente legate a Strix Halo possono restare in uno stack specializzato.

Quelle generalizzabili possono risalire verso llama.cpp, ROCm, PyTorch o altri progetti upstream.

Nel frattempo tool come quelli di Donato possono renderle utilizzabili senza costringere ogni utente a ricostruire manualmente l'intera genealogia delle patch.

---

## Non è solo una storia di LLM

La cosa che mi convince maggiormente che non si tratti di un'anomalia temporanea di llama.cpp è che lo stesso pattern sta comparendo nella generazione di immagini e video.

Con **MiniMax H3**, per esempio, uno dei colli di bottiglia su RDNA3.5 è l'attention su sequenze molto lunghe.

Il percorso generico funziona.

Ma conoscere contemporaneamente la forma precisa dei tensori prodotti dal modello e le caratteristiche di gfx1151 permette di costruire un percorso attention specializzato, evitando trasformazioni e movimenti di dati inutili.

Il risultato interessante non è il numero di secondi risparmiati.

È che qualcuno, al di fuori del team che ha progettato il modello e al di fuori del team che ha progettato la GPU, può studiare entrambi e trovare una [rappresentazione più efficiente del problema](https://github.com/Yasei-no-otoko/ComfyUI-RDNA35-Attention).

Con **Qwen Image 2.1** il concetto arriva ancora più lontano.

Non si interviene soltanto sul kernel.

Si possono combinare attention ottimizzate per l'hardware con tecniche che evitano di ricalcolare completamente parti del modello quando [l'informazione cambia poco tra uno step e il successivo](https://github.com/ciru-ai/ComfyUI-CiruImageAccelerator).

---

## Conclusione: il valore dell'apertura

La cosa che trovo più interessante in tutta questa storia non è stabilire se AMD possa o meno recuperare il vantaggio accumulato da CUDA. È osservare cosa diventa possibile quando modello, runtime e buona parte dello stack software sono aperti. Un problema di prestazioni non appartiene più necessariamente al team che ha progettato la GPU, a quello che ha creato il modello o ai maintainer del framework: chiunque abbia competenze sufficienti può attraversare questi confini, capire dove si perde tempo o banda e provare una soluzione. È esattamente quello che vediamo nei lavori menzionati qui sopra e molti altri in community: idee nate indipendentemente che possono essere studiate, modificate, combinate e verificate da persone che spesso non appartengono alla stessa organizzazione e non si sono mai coordinate a priori.

Per me è questo il valore aggiunto più importante dell'open source. Non semplicemente avere accesso al codice, e nemmeno disporre di sviluppatori che fanno gratuitamente il lavoro che dovrebbe fare il vendor. È creare un ambiente in cui l'innovazione può avvenire **senza dover essere pianificata centralmente**. Un formato numerico nato per risolvere un problema può suggerire un nuovo kernel; un'ottimizzazione scritta per un modello può rivelare una proprietà utile della GPU; qualcuno può prendere entrambe, integrarle in un altro runtime e scoprire qualcosa che nessuno dei progetti originali aveva previsto. Anche gli esperimenti che non sopravvivono lasciano dietro di sé benchmark, codice, discussioni e conoscenza tecnica riutilizzabile.

Naturalmente questo ha un costo: duplicazione degli sforzi, fork incompatibili, esperimenti che si rivelano vicoli ciechi e una user experience che, soprattutto nelle fasi iniziali, è molto lontana dalla prevedibilità di uno stack maturo come CUDA. Ma la frammentazione che vediamo oggi può essere anche il laboratorio dal quale emergeranno le soluzioni di domani. Progetti come halo-box provano già a raccogliere idee nate altrove; kyuz0 le rende più facilmente riproducibili; ciò che si dimostra abbastanza generale può infine arrivare negli upstream. In questo processo non è necessario che sopravvivano tutti i repository: è molto più importante che possano sopravvivere **le idee**.

Da ingegnere che lavora da molto tempo con l'open source, probabilmente guardo questo fenomeno con una certa predisposizione favorevole. Ma Strix Halo mi sembra un caso particolarmente concreto di qualcosa che ho visto molte volte: l'apertura non sostituisce l'investimento del produttore, ma può moltiplicarne gli effetti. AMD fornisce il silicio e sviluppa ROCm; i progetti upstream costruiscono le fondamenta comuni; una comunità distribuita esplora contemporaneamente strade che nessuna singola azienda potrebbe probabilmente prioritizzare tutte. Il risultato è che, mesi dopo aver acquistato una macchina, quello stesso hardware può fare cose che al momento del lancio erano lente, difficili o semplicemente non previste. **Il silicio non è cambiato. È aumentata la conoscenza collettiva di come usarlo.** Ed è forse questa la parte più interessante della storia.

---

## Riferimenti e approfondimenti

- [ROCmFPX — Carlo Pasquale](https://github.com/charlie12345/ROCmFPX)
- [DwarfStar/DS4 — Salvatore Sanfilippo (antirez)](https://github.com/antirez/ds4)
- [Ciru — profilo Hugging Face](https://huggingface.co/jcbtc/activity/community)
- [DualView — Ciru](https://llm.ciru.ai/dualview#explainer)
- [Qwen3.8-27B CIRU ActiveFPX/PromptForge — Hugging Face](https://huggingface.co/jcbtc/Qwen3.8-27B-CIRU-ActiveFPX-PromptForge)
- [Nathan Wilson — fork llama.cpp per Strix Halo](https://github.com/Nathanw1014)
- [EngramHalo — Aristo94](https://github.com/Aristo94/EngramHalo.cpp)
- [Halogen Flash Server — Peonist.ai](https://github.com/peonist-ai/halogen-flash-server)
- [pwilkin — GitHub](https://github.com/pwilkin)
- [AMD Strix Halo Toolboxes — kyuz0 (Donato Capitella)](https://github.com/kyuz0/amd-strix-halo-toolboxes)
- [strix-llama.cpp — halo-box](https://github.com/halo-box/strix-llama.cpp)
- [ComfyUI RDNA35 Attention — Yasei-no-otoko](https://github.com/Yasei-no-otoko/ComfyUI-RDNA35-Attention)
- [ComfyUI CiruImageAccelerator — ciru-ai](https://github.com/ciru-ai/ComfyUI-CiruImageAccelerator)
- [HIP — ROCm documentation](https://rocmdocs.amd.com/projects/HIP/)
- [AMD — Vulkan ecosystem](https://www.amd.com/en/products/graphics/ecosystems/vulkan.html)
- [AMD Ryzen AI Max+ 395 — scheda prodotto](https://www.amd.com/en/products/processors/laptop/ryzen/ai-300-series/amd-ryzen-ai-max-plus-395.html)
- [llama.cpp — ggml-org](https://github.com/ggml-org/llama.cpp)
