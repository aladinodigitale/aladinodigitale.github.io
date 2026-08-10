---
title: "Approfondimento del denoising: scheduler, sampler e low-step LoRA"
date: 2026-08-10
excerpt: "Quando un modello generativo può essere eseguito localmente, diventa possibile guardare dentro il processo di generazione. Vediamo cosa significano davvero scheduler, sampler, sigma shift, steps e low-step LoRA in una pipeline moderna come MiniMax H3."
tags: [AI locale, generative AI, ComfyUI, MiniMax H3, flow matching, LoRA, sampler, scheduler]
author: alessio
classes: wide
header:
  overlay_image: /assets/images/sampler-scheduler-low-step-lora/overlay.jpg
  overlay_filter: 0.5
---

L'uscita di un nuovo modello generativo tende a produrre sempre la stessa sequenza: benchmark, esempi, confronti, prompt, classifiche. Con MiniMax H3, presentato a fine luglio e reso disponibile con i pesi pochi giorni dopo, è successo naturalmente anche questo.

Ma c'è un altro aspetto, secondo me più interessante: quando un modello può essere eseguito localmente e il suo workflow è accessibile, diventa possibile **guardare dentro il processo di generazione**, separarne i componenti e sperimentare.

È quello che sta succedendo in questi giorni attorno a H3. Nel giro di pochissimo tempo sono comparsi workflow ComfyUI, quantizzazioni, implementazioni alternative, sampler specifici e LoRA pensati per ridurre drasticamente il numero di step necessari. Non è tanto H3 in sé il tema di questo post: è piuttosto l'occasione per capire cosa significhino davvero alcune delle voci che troviamo nei workflow — *scheduler*, *sampler*, *sigma shift*, *steps*, *low-step LoRA* — e soprattutto perché intervenire su queste componenti possa migliorare la qualità, ridurre il tempo di generazione, o entrambe le cose.

L'obiettivo non è entrare nei dettagli dell'architettura specifica di H3, dei suoi Transformer, dell'attention o del training. Ci interessa un livello più alto: **come avviene il sampling di una moderna pipeline generativa e dove possiamo intervenire per renderlo più efficiente**.

> Nota terminologica: userò spesso "denoising" nel senso intuitivo di processo iterativo che porta da uno stato rumoroso al contenuto finale. H3 è in realtà un modello basato su *flow matching*, non un diffusion model classico. Vedremo tra poco perché, per il discorso che ci interessa, l'astrazione resta comunque molto utile.

---

## Prima di sampler e scheduler: come nasce un'immagine o un video?

Semplificando molto, una pipeline moderna di generazione può essere vista così:

```text
prompt / immagini / video / audio di riferimento
                    │
                    ▼
          encoder / conditioning
                    │
                    ▼
             latent iniziale
                    │
                    ▼
         ┌───────────────────┐
         │  GENERAZIONE      │
         │    ITERATIVA      │
         │                   │
         │   model forward   │
         │        ↓          │
         │   latent update   │
         │        ↓          │
         │   model forward   │
         │        ↓          │
         │        ...        │
         └───────────────────┘
                    │
                    ▼
              latent finale
                    │
                    ▼
                 VAE
                    │
                    ▼
             immagine / video
```

Il prompt e gli eventuali riferimenti vengono prima trasformati in rappresentazioni che il modello possa usare come *conditioning*. Nel caso di un'immagine o di un video non lavoriamo normalmente direttamente sui pixel: il contenuto viene rappresentato in uno spazio compresso, il **latent space**.

La parte che ci interessa è il blocco centrale.

La generazione non avviene in una singola esecuzione del modello. Si parte da un latent fortemente rumoroso e lo si trasforma progressivamente fino a ottenere un latent che il VAE possa infine decodificare in pixel.

Per un video il concetto è lo stesso, anche se il latent è molto più grande: deve rappresentare non soltanto altezza e larghezza, ma anche il tempo. In H3, inoltre, audio e video vengono generati congiuntamente nello stesso sistema.

Ed è proprio questo loop iterativo a costare molto.

---

## Diffusion e flow matching: abbastanza teoria da non raccontare bugie

Nei diffusion model classici il punto di partenza concettuale è un processo che aggiunge progressivamente rumore ai dati. Il modello impara poi a percorrere in qualche modo il processo inverso. A seconda della parametrizzazione può predire rumore, score, velocity o altre quantità correlate.

Il **flow matching** guarda il problema da una prospettiva leggermente diversa.

Immaginiamo una distribuzione semplice, come il rumore gaussiano, e la distribuzione complessa delle immagini o dei video che vogliamo generare. Tra le due esiste idealmente un percorso continuo. Il modello impara un **campo vettoriale**: dato il punto in cui ci troviamo lungo quel percorso, ci dice in quale direzione dobbiamo muoverci.

In forma molto astratta:

```text
rumore                                      video
  ● ───────────────────────────────────────► ●

       il modello impara "come muoversi"
       lungo questa trasformazione
```

H3 appartiene a questa seconda famiglia.

Questa distinzione è importante dal punto di vista matematico, ma per quello che vogliamo capire possiamo usare un'astrazione comune:

```text
stato corrente
      │
      ▼
   MODELLO
      │
      ▼
predizione di come evolvere lo stato
      │
      ▼
 aggiornamento
      │
      ▼
stato successivo
```

Da qui in avanti ragioneremo soprattutto come se avessimo un modello *flow-based*, perché è il caso di H3 e rende particolarmente intuitivo il ruolo di scheduler e sampler. Molte delle idee, però, si ritrovano anche nelle pipeline diffusion tradizionali.

---

## Sigma: dove siamo lungo il viaggio

Serve ora una coordinata che descriva **a che punto del processo ci troviamo**.

In molte implementazioni questa coordinata viene espressa tramite `sigma`.

L'intuizione utile è:

```text
sigma alto                                sigma basso

rumore  ───────────────────────────────────────► contenuto
 1.0                                               0.0
```

Non bisogna interpretare sigma semplicemente come "quanto rumore tolgo in questo step". È piuttosto un'indicazione del **livello di rumore dello stato corrente**, ovvero della nostra posizione lungo la traiettoria.

A sigma molto alto il contenuto è ancora quasi tutto da determinare. A sigma basso la struttura è già sostanzialmente emersa e ci troviamo vicino al risultato finale.

Il modello riceve quindi qualcosa che, semplificando, assomiglia a:

```text
latent corrente
sigma / timestep
conditioning
```

e produce una predizione.

Nel caso di un flow model possiamo pensarla come una **velocity**, cioè una direzione locale:

> "Se ti trovi qui, a questo punto del percorso, devi muoverti in questa direzione."

Una singola esecuzione completa del modello viene chiamata **forward pass**.

### Timeline, curva dei sigma e griglia di riferimento

Fin qui abbiamo usato `sigma` come coordinata del processo, ma nelle implementazioni compare spesso anche un'altra coordinata: il **timestep** (o, più genericamente, `t`), cioè una posizione lungo la timeline ideale della generazione.

Le due cose non sono necessariamente la stessa cosa.

Possiamo pensare a `t` come a una coordinata astratta lungo il percorso:

```text
t = 1                                      t = 0
rumore  ───────────────────────────────────► contenuto
```

Il *model sampling* definisce poi una relazione:

```text
sigma = f(t)
```

che associa a ogni posizione della timeline il corrispondente livello di rumore. Se disegnassimo `sigma` in funzione di `t`, otterremmo quella che nel seguito chiameremo **curva dei sigma**.

Questa curva non deve essere lineare. Due timestep equidistanti possono quindi corrispondere a valori di sigma molto diversi tra loro.

Nella pratica il computer non rappresenta una curva continua con infiniti punti. Un'implementazione può costruire una **griglia di riferimento** molto più fitta degli step che useremo davvero per generare: una tabella di timestep e dei corrispondenti sigma. In ComfyUI, per i flow model discreti come quello usato da H3, questa griglia contiene normalmente molti più punti — tipicamente 1000 — rispetto ai 4, 8 o 20 step di una generazione.

È importante non confondere le due cose:

```text
griglia di riferimento del modello
moltissimi punti che descrivono t → sigma
                    │
                    ▼
               scheduler
                    │
                    ▼
pochi sigma realmente usati nella generazione
```

Quei molti punti **non sono altrettanti forward del modello**. Servono a descrivere la relazione tra timeline e sigma e, per alcuni scheduler, a fornire i livelli da cui scegliere i punti effettivi di sampling.

Vedremo tra poco che il *sigma shift* interviene proprio sulla funzione `f(t)`, deformando questa curva.

---

## Il vero costo: quante volte eseguiamo il modello?

Questo è uno dei concetti più importanti dell'intero discorso.

Supponiamo che la generazione usi 20 valutazioni del modello:

```text
sigma 1.00
    │
    ├── model forward
    ▼
sigma ...
    │
    ├── model forward
    ▼
sigma ...
    │
    ├── model forward
    ▼
   ...
    ▼
sigma 0.00
```

Ogni forward significa attraversare l'intero denoiser/Transformer con un latent che, soprattutto nel caso di un video, può essere enorme.

Per questo una prima approssimazione molto utile è:

```text
tempo di sampling ≈ NFE × costo di un forward
```

Una singola esecuzione del modello è una **function evaluation (FE)**, che nel nostro discorso coincide con un forward del denoiser. **NFE** significa invece *Number of Function Evaluations*: è il **numero totale** di FE richieste dalla generazione.

Quindi, per esempio:

```text
NFE = 20  →  il denoiser viene valutato 20 volte
```

Non sempre `steps = NFE`: alcuni sampler possono richiedere più di una FE per uno stesso step. Per confrontare davvero il costo di due strategie di sampling, quindi, **NFE** è più significativo del semplice numero di step mostrato nell'interfaccia.

A questo punto emerge il problema fondamentale:

> La traiettoria ideale è continua, ma noi possiamo permetterci di interrogare il modello soltanto un numero limitato di volte.

Se abbiamo un budget di 20, 8 o addirittura 4 forward, **dove conviene farli? E come usiamo le predizioni ottenute per muoverci da un punto al successivo?**

Sono esattamente le domande a cui rispondono scheduler e sampler.

---

## Scheduler: dove spendere il budget di calcolo

Abbiamo ora due cose:

1. una relazione `t → sigma`, definita dal *model sampling*;
2. un budget limitato di valutazioni del modello.

Lo **scheduler** deve trasformare queste informazioni nella piccola sequenza di sigma che useremo davvero durante la generazione.

Supponiamo, per semplicità, che la relazione tra timeline e sigma fosse lineare e di voler fare pochi step. Potremmo ottenere una sequenza come:

```text
1.00 ─── 0.75 ─── 0.50 ─── 0.25 ─── 0.00
```

Ma non è affatto detto che distribuire le valutazioni in questo modo sia la scelta migliore. Potrebbe essere più utile concentrare i punti in una certa zona del percorso:

```text
1.00 ─ 0.96 ─ 0.87 ───── 0.55 ───────── 0.00
```

oppure fare quasi il contrario.

In ComfyUI, nomi come:

- `simple`
- `karras`
- `exponential`
- `beta`
- `normal`

indicano strategie diverse per costruire questa sequenza.

L'output dello scheduler è quindi essenzialmente:

```text
sigma_0, sigma_1, sigma_2, ... sigma_n
```

Con `steps = 4`, per esempio, ci saranno normalmente quattro intervalli da percorrere e un sigma terminale pari a zero.

### Cosa significa allora `simple`?

Nel caso di ComfyUI il nome è molto letterale.

Lo scheduler `simple` guarda la **griglia di riferimento dei sigma del modello** che abbiamo appena introdotto e ne prende un certo numero di punti a intervalli approssimativamente regolari negli indici, aggiungendo poi `sigma = 0` come termine finale.

Quindi `simple` non significa che il processo di denoising nel suo complesso sia "semplice", e soprattutto non ci dice ancora **come** verrà aggiornato il latent tra un sigma e il successivo.

Dice soltanto, in sostanza:

> "Dato il numero di step richiesto, scegli in modo semplice i livelli di sigma ai quali faremo le valutazioni."

Altri scheduler costruiscono la sequenza con criteri differenti. Karras ed exponential, per esempio, distribuiscono i sigma secondo specifiche trasformazioni matematiche del range; `beta` privilegia certe regioni della griglia secondo una distribuzione beta; `normal` lavora sulla timeline del modello e la riconverte in sigma.

Per il momento ci basta il principio generale:

> **lo scheduler decide dove, lungo la traiettoria, spendere le nostre FE.**

Non abbiamo ancora stabilito come usare la predizione ottenuta in quei punti per arrivare al successivo. Quello sarà il compito del sampler.

---

## Sigma shift: cambiare la geometria della timeline

A complicare — o rendere più interessante — il quadro c'è il **sigma shift**.

Ora possiamo descriverlo in modo più preciso: lo shift interviene sulla relazione `sigma = f(t)` che abbiamo introdotto prima. **Non sceglie gli step**; modifica il valore di sigma associato ai diversi punti della timeline.

Nel caso del flow sampling usato da H3 in ComfyUI, la trasformazione ha la forma:

```text
sigma(t) = shift × t / (1 + (shift - 1) × t)
```

Con `shift = 1` la relazione è lineare: `sigma = t`. Aumentando lo shift, gran parte della timeline viene invece mappata verso sigma più alti.

Possiamo quindi immaginare la catena così:

```text
timeline t
      │
      │  model sampling + sigma shift
      ▼
relazione / curva t → sigma
      │
      │  discretizzata nella griglia di riferimento
      ▼
   scheduler
      │
      ▼
pochi sigma effettivamente usati
```

Uno shift forte può far sì che molti punti della timeline corrispondano ancora a sigma relativamente alti. Per esempio, in un modello con shift elevato, una sequenza apparentemente uniforme sulla timeline può produrre valori del tipo:

```text
1.000 → 0.973 → 0.923 → 0.800 → 0.000
```

anziché:

```text
1.000 → 0.750 → 0.500 → 0.250 → 0.000
```

Il concetto importante non è il valore specifico: è che **la relazione tra "posizione nello step" e livello di rumore non deve essere necessariamente lineare**.

Scheduler e sigma shift sono quindi concettualmente vicini: entrambi influenzano *dove* vengono spese le valutazioni del modello. Non sono però equivalenti.

Il sigma shift fa parte della parametrizzazione della curva di sampling del modello; lo scheduler decide come campionare un numero finito di punti su quella curva.

Ed entrambi possono contribuire allo stesso obiettivo pratico: usare meglio ogni costosa valutazione del denoiser.

---

## Sampler: come ci muoviamo tra due punti

Ora abbiamo una sequenza:

```text
sigma_0 → sigma_1 → sigma_2 → ... → 0
```

A ciascun punto chiediamo al modello una direzione.

Resta però da decidere **come usare quella direzione per aggiornare il latent**.

Questo è il compito del **sampler**, o più precisamente del solver numerico.

### Euler: segui la direzione corrente

Il caso più intuitivo è Euler.

Il modello dice:

```text
sei qui ● ─────────► vai in questa direzione
```

e il sampler compie un passo verso il sigma successivo usando quella previsione.

In forma concettuale:

```text
prediction = model(x, sigma)
x_next = x + prediction × step_size
```

È semplice ed economico: tipicamente una valutazione del modello per ogni intervallo.

Il limite è altrettanto intuitivo. Se la traiettoria curva molto, una direzione misurata nel punto di partenza diventa meno accurata quanto più grande è il passo.

### Heun e i metodi che verificano la traiettoria

Un metodo di ordine superiore può fare qualcosa del genere:

1. chiede al modello la direzione corrente;
2. calcola dove arriverebbe;
3. interroga nuovamente il modello nel punto stimato;
4. combina le due predizioni per ottenere un aggiornamento migliore.

Il risultato numerico può essere più accurato, ma abbiamo pagato due forward invece di uno.

Questo evidenzia perché **contare soltanto gli step può essere fuorviante**.

### Multistep: ricordarsi il passato

I metodi *multistep* adottano una strategia particolarmente interessante per i grandi modelli generativi.

Invece di interrogare più volte il denoiser nello stesso intervallo, conservano le predizioni ottenute negli step precedenti:

```text
sigma A → prediction A
sigma B → prediction B
sigma C → prediction C
```

e usano questa storia per stimare meglio come sta cambiando la traiettoria.

L'obiettivo è ottenere un'integrazione più accurata senza necessariamente raddoppiare il numero di costosi forward.

Questo è il principio che rende sampler multistep particolarmente interessanti quando il denoiser è enorme: **ottenere più valore da ogni singola FE**.

---

## DPM++, ancestral, SDE: perché esistono così tanti sampler?

Le dropdown di ComfyUI possono diventare rapidamente intimidatorie:

```text
euler
euler_ancestral
heun
dpmpp_2m
dpmpp_2m_sde
dpmpp_3m_sde
res_multistep
...
```

Non sono semplicemente varianti "più o meno buone" dello stesso algoritmo.

Cambiano diverse proprietà:

- l'ordine del solver;
- il fatto che sia single-step o multistep;
- quante valutazioni del modello richieda;
- se il processo sia deterministico;
- se venga reintrodotta stocasticità durante il sampling;
- come vengano trattate le predizioni precedenti.

Un sampler *ancestral* o SDE, per esempio, può reintrodurre rumore durante il percorso. Questo può essere utile per varietà e texture, ma per un video la stocasticità deve fare i conti anche con la coerenza temporale.

Non esiste quindi "il sampler migliore" in assoluto.

Esiste un sampler più o meno adatto a:

- quel modello;
- quello scheduler;
- quel numero di step;
- quel tipo di contenuto;
- il compromesso qualità/tempo che vogliamo ottenere.

---

## Ridurre gli step: perché non basta semplicemente scrivere 4

A questo punto potrebbe sembrare che la soluzione sia banale: se 20 forward costano troppo, impostiamo `steps = 4`.

Il problema è che con quattro valutazioni i salti diventano enormi.

Immaginiamo una traiettoria curva:

```text
traiettoria corretta

A ●
    \
     \
      ●
       \
        \____ ● B
```

Con tanti step piccoli una previsione locale leggermente imprecisa viene presto corretta dalla chiamata successiva.

Con quattro step, invece, potremmo chiedere al modello di fare qualcosa del genere:

```text
A ● ───────────────────────► B'
```

Una piccola imprecisione nella direzione iniziale produce un errore molto più grande alla fine del salto.

Scheduler e sampler possono aiutarci a scegliere punti migliori e integrarli meglio, ma a un certo punto emerge un limite: **il modello stesso non è stato necessariamente addestrato per lavorare bene con una discretizzazione così aggressiva**.

È qui che entrano in gioco i low-step LoRA.

---

## Low-step LoRA: questa volta cambiamo il modello

Un **LoRA** modifica il comportamento del modello aggiungendo adattamenti a basso rango ai suoi pesi. È un meccanismo molto più leggero rispetto alla distribuzione di un checkpoint completo.

Nel caso dei LoRA per *few-step inference*, l'obiettivo non è cambiare lo stile del video o insegnare un nuovo personaggio. L'obiettivo è modificare il denoiser affinché produca predizioni utili **quando il sampling usa pochissimi step**.

Il contrasto con scheduler e sampler è importante.

Scheduler:

> **Dove** interrogo il modello?

Sampler:

> **Come** uso le sue predizioni per muovermi?

Low-step LoRA:

> Come modifico **il modello stesso** perché le sue predizioni restino buone quando lo interrogo molto meno spesso?

Un LoRA di questo tipo viene tipicamente addestrato o distillato per uno specifico regime few-step. Per questo spesso arriva accompagnato da indicazioni precise sul numero di step, sullo scheduler e sul sampler da utilizzare.

Non è un generico interruttore "turbo".

Se un LoRA è stato ottimizzato, per esempio, per una certa sequenza di quattro sigma, cambiare arbitrariamente scheduler significa chiedere al modello di lavorare fuori dalla configurazione per la quale è stato adattato.

Questa è una differenza importante rispetto a una normale ottimizzazione implementativa: **qui qualità e velocità vengono scambiate già durante l'addestramento/adattamento del modello**.

---

## Un workflow reale: MiniMax H3 in ComfyUI

Ed eccoci finalmente al motivo per cui H3 è un esempio così utile.

Un workflow ComfyUI per H3 con un Turbo LoRA può contenere contemporaneamente questi blocchi:

```text
Load LoRA
    │
    ▼
ModelSamplingMiniMaxH3
sigma shift
    │
    ▼
BasicScheduler
scheduler = simple
steps = 4
    │
    │ produce i sigma
    ▼
SamplerCustomAdvanced ◄──── KSamplerSelect
                          res_multistep
```

{% include figure image_path="/assets/images/sampler-scheduler-low-step-lora/comfyui-workflow.jpg" alt="Workflow ComfyUI per MiniMax H3 con Turbo LoRA" caption="Un workflow ComfyUI per MiniMax H3 con Turbo LoRA: model sampling, scheduler, sampler e campionatore avanzato cooperano per generare in soli 4 step." %}

A prima vista può sembrare che `simple`, `res_multistep`, `sigma shift` e `4 steps` siano quattro modi diversi di controllare la stessa cosa.

In realtà sono quattro pezzi distinti che cooperano.

### 1. Il Turbo LoRA

Modifica H3 affinché possa lavorare in un regime few-step.

### 2. `ModelSamplingMiniMaxH3`

Imposta la particolare relazione tra timeline e sigma utilizzata dal modello. H3 gestisce inoltre video e audio su schedule differenti, pur generandoli congiuntamente.

### 3. `BasicScheduler: simple`

Prende la curva dei sigma risultante e seleziona i punti da utilizzare per i quattro step.

Il risultato non è necessariamente una sequenza lineare di sigma: `simple` lavora sulla griglia del modello, che è già stata deformata dal sigma shift.

### 4. `KSamplerSelect: res_multistep`

Non produce sigma.

Riceve i sigma scelti dallo scheduler e decide come integrare il percorso tra un punto e l'altro, utilizzando anche informazioni provenienti dalle predizioni precedenti.

Infine `SamplerCustomAdvanced` mette insieme:

```text
noise iniziale
conditioning
modello
sampler
lista dei sigma
```

ed esegue effettivamente il loop.

Visto così, il workflow smette di essere una collezione di dropdown misteriose e diventa la rappresentazione esplicita di un algoritmo numerico.

---

## Tre modi diversi di rendere più veloce la generazione

A questo punto possiamo organizzare le ottimizzazioni in modo abbastanza pulito.

### 1. Ridurre il costo di ogni forward

Per esempio:

- ridurre risoluzione;
- ridurre il numero di frame;
- quantizzare i pesi;
- usare kernel di attention più efficienti;
- sfruttare sparse attention;
- migliorare l'implementazione hardware.

In questo caso:

```text
NFE invariato
×
forward più economico
=
generazione più veloce
```

### 2. Fare meno forward

È il territorio dei modelli few-step e dei low-step LoRA:

```text
NFE: 20 → 8 → 4
```

Se il costo del denoiser domina il tempo totale, la riduzione può essere enorme.

### 3. Ottenere più valore da ogni forward

Qui entrano scheduler e sampler.

Un migliore posizionamento dei sigma e un solver più adatto possono produrre:

- più qualità a parità di NFE;
- qualità simile riducendo NFE.

Ed è per questo che qualità e velocità non sono problemi separati.

---

## L'obiettivo vero: quality per compute

Dire "questo sampler è più veloce" è spesso una semplificazione sbagliata.

Se due sampler hanno entrambi `NFE = 20`, il tempo trascorso nel denoiser sarà grossomodo simile. Uno dei due potrebbe però produrre una qualità migliore.

A quel punto potremmo scoprire che con il sampler migliore basta `NFE = 12` per raggiungere la qualità che l'altro otteneva con `NFE = 20`.

**È lì che nasce il vero speed-up.**

Lo stesso ragionamento vale per uno scheduler.

E per un low-step LoRA il concetto viene spinto ancora oltre: adattiamo il modello affinché possa estrarre molto più risultato da pochissime valutazioni.

La metrica mentale più utile diventa quindi qualcosa del tipo:

```text
                qualità ottenuta
efficienza = ───────────────────────
              compute necessario
```

Non stiamo cercando semplicemente di rendere il timer più piccolo. Stiamo cercando di ottenere il miglior risultato possibile per una quantità limitata di compute.

---

## Quattro step non significano automaticamente cinque volte più veloce

C'è però un'ultima precisazione importante.

Se il sampling passa da `NFE = 20` a `NFE = 4`, **la parte di denoising** può teoricamente avvicinarsi a un'accelerazione di 5×.

Ma la pipeline completa comprende anche:

```text
text / vision encoding
conditioning
preparazione dei latent
sampling
VAE video decode
VAE audio decode
encoding/mux del video finale
```

Questi costi non scompaiono riducendo gli step.

Quindi:

```text
tempo totale =
    costi fissi
  + NFE × costo_forward
  + decoding/output
```

Più il denoiser domina il runtime, più la riduzione di NFE si avvicina allo speed-up teorico. Più sono importanti encoder e VAE, meno il guadagno complessivo sarà proporzionale.

Anche questo è un motivo per cui, parlando di performance, è utile distinguere **sampling time** e **end-to-end generation time**.

---

## Il valore di poter vedere il workflow

Ed è qui che torniamo, solo alla fine, all'apertura del modello.

Se utilizziamo un generatore video esclusivamente attraverso un'API, vediamo qualcosa di molto semplice:

```text
prompt → video
```

Il provider può utilizzare internamente distillazione, scheduler particolari, solver proprietari, caching, sparse attention o qualsiasi altra ottimizzazione. Per noi il processo rimane una scatola nera.

Con pesi eseguibili localmente e strumenti modulari come ComfyUI vediamo invece:

```text
conditioning
     ↓
model sampling
     ↓
scheduler
     ↓
sampler
     ↓
latent
     ↓
VAE
```

E possiamo intervenire.

Non perché ogni utente debba inventarsi un nuovo solver numerico, ma perché la possibilità di osservare e modificare il workflow rende molto più facile **capire cosa sta realmente succedendo**.

H3 è semplicemente l'esempio del momento. La community ha avuto accesso al modello, ComfyUI ha reso visibili i pezzi della pipeline e nel giro di pochi giorni sono partiti gli esperimenti: nuovi workflow, quantizzazioni, few-step LoRA, scheduler e sampler alternativi.

Ed è proprio questo il punto più interessante.

L'AI aperta non significa soltanto poter scaricare un file molto grande invece di chiamare un'API.

Significa anche poter aprire il cofano, seguire i cavi e scoprire che dietro un apparentemente banale pulsante **Generate** c'è un problema molto concreto di calcolo numerico:

> abbiamo una traiettoria continua, un numero limitato di costose valutazioni del modello e dobbiamo decidere **dove farle, come usarle e quanto possiamo permetterci di distanziarle**.

Una volta capito questo, `simple`, `Karras`, `res_multistep`, `4 steps` e `Turbo LoRA` smettono di sembrare formule magiche.

Sono semplicemente strumenti diversi per rispondere a quelle tre domande.

---

## Riferimenti e approfondimenti

- [MiniMax — MiniMax H3 open-source announcement](https://www.minimax.io/news/minimax-h3-open-source)
- [MiniMax H3 — repository ufficiale](https://github.com/MiniMax-AI/MiniMax-H3)
- [ComfyUI — workflow MiniMax H3](https://docs.comfy.org/tutorials/video/minimax/minimax-h3)
- [Flow Matching for Generative Modeling — Lipman et al.](https://arxiv.org/abs/2210.02747)
- [ComfyUI — implementazione degli scheduler](https://github.com/Comfy-Org/ComfyUI/blob/master/comfy/samplers.py)
- [ComfyUI — nodi MiniMax H3 e sigma shift](https://github.com/Comfy-Org/ComfyUI/blob/master/comfy_extras/nodes_minimax_h3.py)
- [MiniMax H3 Turbo LoRA — esempio community few-step](https://huggingface.co/larryvrh/MiniMax-H3-Turbo-Lora)
