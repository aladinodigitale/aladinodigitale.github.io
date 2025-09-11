---
title: "Il desiderio perfetto: come il prompt cambia tutto (Krea 1 vs Gemini 2.5 Flash Image “Nano Banana”)"
date: 2025-09-11 10:00:00 +0200
tags: [prompt engineering, image generation, Gemini, Krea]
description: "Desiderio vago → il genio interpreta. Desiderio-contratto → il genio esegue. Stesso scenario, 5 versioni di prompt, due modelli: cosa cambia davvero."
---

> **Metafora della lampada** — *Desiderio vago → il genio interpreta. Desiderio-contratto → il genio esegue.*  
> In questo articolo non c’è un “vincitore”: osserviamo come **Krea 1** e **Gemini 2.5 Flash Image (Nano Banana)** rispondono a desideri formulati in modo diverso.

## Perché questo post
Quando si parla di generazione di immagini, la qualità del risultato dipende tanto dal modello utilizzato quanto da **come formuli il prompt**. Ho messo a confronto **Krea 1** e **Gemini 2.5 Flash Image** su una sequenza di prompt progressivamente più “rigorosi”, mantenendo lo **stesso scenario**. L’obiettivo è mostrare **cosa cambia** tra un desiderio vago e un vero **contratto operativo**.

> Nota lingua: al momento del test, **Krea** ha risposto meglio in **inglese**, quindi i prompt sono in inglese.

## Metodo
1. **Scenario unico**: ritratto editoriale di una giovane designer nella **metro di Milano**.  
2. **5 versioni di prompt (V1→V5)**: da **vago** → **contratto** → **referenza** → **serie** → **editing**.  
3. **Due modelli**: **Krea 1** (krea.ai) e **Gemini 2.5 Flash Image (Nano Banana)**.

---

## I prompt (da copiare)

### V1 — Minimal (vago)
```text
Urban portrait of a young female designer in the Milan subway, realistic style, neon lighting.
```

### V2 — Prompt-contratto (strutturato)
```text
INTENT: editorial, realistic portrait suitable for a magazine cover.
SUBJECT: young Italian woman, chin-length brown bob haircut, thin metal glasses, navy-blue blazer.
SCENE: corridor of the Milan metro, advertising billboards in the background deliberately blurred (background bokeh).
CAMERA: medium shot, 35mm look, f/2.0, pronounced bokeh.
LIGHT: cool neon from above, subtle reflections on a slightly wet floor.
PALETTE/STYLE: muted teal-and-orange, clean editorial finish.
ACTION: she holds a rigid portfolio with the word “ALADINO” clearly readable.
CONSTRAINTS: hands fully visible; no other people; render the text “ALADINO” exactly (ALL CAPS, no misspellings).
POST: fine grain.
```

### V3 — Contratto + referenza (consistenza del soggetto)
*(Allega una reference della protagonista, ad es. il miglior output di V2 o un tuo riferimento.)*
```text
Use the attached image as a REFERENCE to preserve the character’s face identity and hairstyle.
Repeat the CONTRACT prompt from V2 and keep the SAME person: same facial features, glasses, hair length and color.
Match skin tone and lighting to the reference if needed. Everything else stays as in V2.
```

### V4 — Serie coerente (4 scatti, stessa identità)
*(Opzione migliore: allega come reference la migliore V2/V3. Se l’interfaccia non supporta reference multiple, esegui 4 edit sequenziali della stessa immagine cambiando solo l’angolo.)*
```text
Create 4 images of the SAME protagonist (identical face identity and outfit as in the reference), keeping lighting, palette, and environment unchanged (Milan metro corridor, blurred billboards).

Angles:
A) frontal medium shot
B) three-quarter (3/4) medium shot
C) profile (side) medium shot
D) low-angle medium shot

Constraints: hands visible in A and B; the word “ALADINO” must be clearly readable and correctly spelled in at least A and B; no other people.
```

### V5 — Editing linguistico (modifica mirata su una immagine della serie)
*(Seleziona la migliore immagine di V4 e usa Edit/Image-to-Image.)*
```text
Change ONLY this detail: replace the rigid portfolio with a black tote bag featuring the text “ALADINO” in white, clearly readable and without misspellings.
Preserve the same protagonist, framing, lighting, palette, depth of field, and environment.
```

---

## Risultati & commento (V1 → V5)
> **Non è una classifica.** Mettiamo in evidenza *cosa succede* a parità di desiderio.

### V1 — Prompt vago → il genio **interpreta**
- **Krea 1**: sorprendente fotorealismo e **velocità** (più varianti in pochi secondi). Il prompt generico lascia ampio spazio d’interpretazione: in alcuni scatti la scena è su un **treno** anziché nel corridoio; i **testi** sui cartelli luminosi restano imprecisi.
<a href="/assets/images/2025-09-prompt-krea-gemini/krea-v1-a.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/krea-v1-a.png" alt="Krea - Prompt V1 - es 1" />
</a>

<a href="/assets/images/2025-09-prompt-krea-gemini/krea-v1-b.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/krea-v1-b.png" alt="Krea - Prompt V1 - es 2" />
</a>

<a href="/assets/images/2025-09-prompt-krea-gemini/krea-v1-c.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/krea-v1-c.png" alt="Krea - Prompt V1 - es 3" />
</a>


- **Gemini (Nano Banana)**: immagini sostanzialmente fotorealistiche; stile a tratti lievemente **cinematic/punk** . La generazione è **una alla volta**; talvolta compaiono elementi poco plausibili (es. una panchina in posizione improbabile): effetto collaterale del prompt generico.
<a href="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v1-a.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v1-a.png" alt="Gemini - Prompt V1 - es 1" />
</a>

<a href="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v1-b.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v1-b.png" alt="Gemini - Prompt V1 - es 2" />
</a>

<a href="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v1-c.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v1-c.png" alt="Gemini - Prompt V1 - es 3" />
</a>

<a href="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v1-d.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v1-d.png" alt="Gemini - Prompt V1 - es 4" />
</a>

---

### V2 — Prompt-contratto → il genio **esegue**
- **Krea 1**: buona aderenza al contratto; **ALADINO** ben leggibile sul portfolio; bokeh credibile e integrazione soggetto/sfondo convincente.
<a href="/assets/images/2025-09-prompt-krea-gemini/krea-v2-a.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/krea-v2-a.png" alt="Krea - Prompt V2 - es 1" />
</a>

<a href="/assets/images/2025-09-prompt-krea-gemini/krea-v2-b.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/krea-v2-b.png" alt="Krea - Prompt V2 - es 2" />
</a>

<a href="/assets/images/2025-09-prompt-krea-gemini/krea-v2-c.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/krea-v2-c.png" alt="Krea - Prompt V2 - es 3" />
</a>


- **Gemini**: aderente al contratto con più **varianza** nelle espressioni e negli sfondi; a volte un leggero “**effetto compositing**”.
<a href="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v2-a.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v2-a.png" alt="Gemini - Prompt V2 - es 1" />
</a>

<a href="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v2-b.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v2-b.png" alt="Gemini - Prompt V2 - es 2" />
</a>

<a href="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v2-c.png" target="_blank">
  <img src="/assets/images/2025-09-prompt-krea-gemini/gemini-nb-v2-c.png" alt="Gemini - Prompt V2 - es 3" />
</a>

---

### V3 — Referenza identità → **coerenza** tra generazioni
Ho generato un'immagine di riferimento (reference) utilizzando ChatGPT; l'immagine non è particolarmente ben riuscita ma volutamente ritrae una scena differente ed di proposito è ottenuta da un altro modello.
![Reference V3](</assets/images/2025-09-prompt-krea-gemini/chatgpt-ref-v3.png>)

- **Krea 1**: la reference può dominare il risultato e introdurre **allucinazioni** (es. scritta **ALLADINO** al collo della donna, **Duomo** sullo sfondo con caratteristiche alterate). Si capisce come Krea, basato su Flux, non abbia le funzionalità di editing avanzato di Flux Kontext.
![Krea V3](</assets/images/2025-09-prompt-krea-gemini/krea-v3-a.png>)

![Krea V3](</assets/images/2025-09-prompt-krea-gemini/krea-v3-b.png>)


- **Gemini**: mantiene **identità e hairstyle** della protagonista in modo coerente, rispettando gli altri vincoli del contratto.

![Gemini V3](</assets/images/2025-09-prompt-krea-gemini/gemini-nb-v3-a.png>)


---

### V4 — Serie a 4 inquadrature → **regia** e **pose**
Ho selezionato la prima delle immagini generate da Krea in V2 e l'ho fornita come riferimento per le altre inquadrature.

- **Krea 1**: genera ritratti fotorealistici ma tende a restare su **inquadrature frontali**; non rispetta **angoli/perspective** richiesti. In un caso, parte dell’outfit appare mancante (maglietta/camicia assente sotto il blazer).
![Krea V4](</assets/images/2025-09-prompt-krea-gemini/krea-v4-a.png>)

- **Gemini**: genera egregiamente le **4 inquadrature** richieste (frontale, 3/4, profilo, low-angle); si tratta di uno dei punti di forza del modello. Impressionante lavoro su abbigliamento, movimento delle braccia, espressioni del viso, etc.

![Gemini V4](</assets/images/2025-09-prompt-krea-gemini/gemini-nb-v4-a.png>)

![Gemini V4](</assets/images/2025-09-prompt-krea-gemini/gemini-nb-v4-b.png>)

![Gemini V4](</assets/images/2025-09-prompt-krea-gemini/gemini-nb-v4-c.png>)

![Gemini V4](</assets/images/2025-09-prompt-krea-gemini/gemini-nb-v4-d.png>)


---

### V5 — Editing mirato → **chirurgia** sull’oggetto
- **Krea 1**: di nuovo, come ci si aspettava, Krea continua a generare immagini molto belle, ma non riesce ad effettuare la modifica richiesta. In tre casi su quattro viene aggiunta una borsa, senza rimuovere il portfolio.
![Krea V5](</assets/images/2025-09-prompt-krea-gemini/krea-v5-a.png>)

![Krea V5](</assets/images/2025-09-prompt-krea-gemini/krea-v5-b.png>)

![Krea V5](</assets/images/2025-09-prompt-krea-gemini/krea-v5-c.png>)


- **Gemini**: esegue l’editing in modo **minimale e preciso**: sostituisce il portfolio con la tote bag, mantenendo tutto il resto invariato.

![Gemini V5](</assets/images/2025-09-prompt-krea-gemini/gemini-nb-v5-a.png>)


---

## Cosa ci portiamo a casa
- **Desiderio vago** → i modelli “riempiono i vuoti”: Krea tende al **fotorealismo** rapido; Gemini a un **look cinematico** coerente.  
- **Desiderio-contratto** → entrambi migliorano; **Gemini** mostra vantaggi su **serie coerenti** ed **edit puntuali**; **Krea** brilla per **impatto fotorealista** e velocità.  
- **Reference & identity** → **Gemini** offre più controllo multi-immagine; **Krea** può sovrainiettare elementi della referenza.  

> Sintesi: **progetta il desiderio**. Se vuoi *variazioni creative e fotorealismo veloce*, Krea è una bella lampada. Se vuoi *serie coerenti ed edit chirurgici*, Gemini è un genio affidabile.

---

## Template riusabile: **Prompt = Contratto**
In questo articolo abbiamo presentato una proposta di prompt strutturato. Ad onor del vero, la documentazione delle API di Gemini pubblicata da Google copre molteplici aspetti, tra i quali si le [best practice per la scrittura dei prompt] (https://ai.google.dev/gemini-api/docs/image-generation?hl=it#template_10) (comunque sempre di tipo discorsivo).
```text
[INTENT] = output e realismo (editoriale, illustrativo, ecc.)
[SUBJECT] = identità, età, tratti, outfit, segni distintivi
[SCENE] = luogo, epoca, oggetti chiave, relazioni
[CAMERA] = piano (full/mezzo busto), 24/35/85mm, f/stop, DoF
[LIGHT] = tipo, qualità, direzione
[PALETTE/STYLE] = palette, mood, riferimenti
[ACTION] = verbo chiaro (stringe/versa/indica…)
[CONSTRAINTS] = mani visibili, testo esatto, niente persone extra
[POST] = grana, vignetta, eventuali ritocchi
```

### Come replicare
1) Copia i prompt qui sopra e ripeti **V1→V5** su ogni modello.  
2) Per **V3/V4** allega sempre la **stessa referenza**.  
3) Per **V5** usa **Edit/Image-to-Image** sulla migliore immagine della serie.

