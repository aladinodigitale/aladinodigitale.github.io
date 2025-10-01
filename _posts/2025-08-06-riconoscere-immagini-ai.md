---
title: "Come riconoscere un'immagine generata da AI"
date: 2025-08-06
categories: [AI, watermarking, C2PA]
tags: [SynthID, AI, watermarking, C2PA, chatgpt]
author: alessio
classes: wide
header:
  overlay_image: /assets/images/chatgpt-woman-green-gimp.jpg
  overlay_filter: 0.8
---
---
Con la diffusione esplosiva delle immagini generate da intelligenze artificiali come DALL·E, Midjourney, Firefly o Imagen, diventa sempre più importante **capire se un contenuto visivo è autentico o sintetico**.  
Non per fare i detective digitali… ma per **trasparenza, responsabilità e sicurezza**.

## 💡 Perché serve distinguere le immagini AI

Ecco alcune motivazioni concrete:

- **Lotta alla disinformazione**: un’immagine deepfake può alimentare fake news virali.
- **Tutela dell’identità e del copyright**: sapere se qualcosa è “vero” aiuta a proteggere contenuti e persone.
- **Contesto etico e narrativo**: in un blog, un giornale o un portfolio, è giusto chiarire cosa è generato da AI.
- **Conformità alle policy**: social network, motori di ricerca e siti di stock stanno chiedendo sempre più spesso **contenuti etichettati**.

## 🏷️ Metadati: la prima linea di difesa

Un’immagine può contenere nel file stesso alcune informazioni utili:

- Nome del software che l’ha generata (`SoftwareAgent`)
- Data di creazione
- Modello AI, piattaforma di generazione, licenza, ecc.

Questi sono **metadati standard**: EXIF, IPTC o XMP.  
Purtroppo, però, sono **facilmente modificabili o rimovibili**.  
Basta aprire l’immagine in un editor come GIMP e salvarla: i metadati spesso **spariscono**.

## 🔐 C2PA e Content Credentials: un nuovo standard per la trasparenza

Per migliorare la fiducia nei contenuti digitali, Adobe, Microsoft, Google, NYT e altri hanno creato la specifica **C2PA (Coalition for Content Provenance and Authenticity)**.

Questa tecnologia consente di **firmare digitalmente un “manifest”** che accompagna l’immagine. Questo manifest contiene:

- l’origine dell’immagine (es. “generata da AI con DALL·E”)
- le modifiche fatte (es. “ritagliata con Photoshop”)
- una firma digitale verificabile

I file così sigillati possono essere **verificati pubblicamente** con strumenti come [contentcredentials.org/verify](https://contentcredentials.org/verify).

### 🧪 Il mio test: metadati con e senza GIMP

Ho generato un’immagine PNG con ChatGPT (modello DALL·E 3), che integra automaticamente **Content Credentials** firmate da OpenAI.

![chatgpt-woman-green.png](/assets/images/chatgpt-woman-green.png)

A seguire, ho caricato l'immagine su [contentcredentials.org/verify](https://contentcredentials.org/verify) ed essa è stata correttamente riconosciuta come "AI generated" di origine "ChatGPT" (il tutto firmato da OpenAI).

![Verifica su contentcredentials.org](/assets/images/cr-check_chatgpt.png)  

Poi ho aperto la stessa immagine con **GIMP** su Linux e l’ho riquadrata e risalvata in formato JPEG.  

![chatgpt-woman-green-gimp.jpg](/assets/images/chatgpt-woman-green-gimp.jpg)

La stessa verifica con l'immagine nuova mostra come i metadati C2PA siano completamente scomparsi.
![Verifica immagine passata in GIMP](/assets/images/cr-check_gimp.png)  

## 🌊 Watermark invisibili nei pixel

Un approccio più resistente è quello dei **watermark invisibili**, ovvero delle modifiche sottili ai pixel che **non alterano l’aspetto visivo**, ma possono essere rilevate da software specializzati.

### Le principali tecniche usate:

### 📐 DCT (Discrete Cosine Transform)

La DCT è alla base della compressione JPEG.  
Consente di rappresentare le immagini come una somma di componenti frequenziali (blocchi 8×8).  
Modificando leggermente i **coefficienti di frequenze basse o medie**, si può nascondere un messaggio che:

- è invisibile a occhio nudo
- **resiste alla compressione JPEG**
- può essere estratto tramite trasformata inversa

🔗 [Approfondimento sulla DCT per watermarking](https://en.wikipedia.org/wiki/Discrete_cosine_transform#Applications)

### 🌊 Wavelet

I wavelet rappresentano l’immagine a più risoluzioni (multi-scala).  
Permettono di inserire watermark in **regioni specifiche**, più stabili all’editing.

- Più resistente a filtri e modifiche localizzate
- Utile per watermark **robusti ma meno densi**

🔗 [Pubblicazione scientifica sul watermarking wavelet-based](https://www.researchgate.net/publication/24346540_Wavelet-based_digital_image_watermarking)

### 🕵️‍♂️ Steganografia (es. LSB)

Le tecniche steganografiche modificano i **bit meno significativi (LSB)** di ciascun pixel.  
Ideale per nascondere messaggi binari o firme:

- molto semplice da implementare
- **poco resistente alla compressione o ai ritagli**
- facilmente distruttibile, ma difficile da notare

🔗 [Tecniche LSB su Wikipedia](https://en.wikipedia.org/wiki/Least_significant_bit)

## 🧠 SynthID: la proposta invisibile di Google

Google DeepMind ha sviluppato **SynthID**, una tecnologia per inserire **watermark invisibili** nelle immagini generate da AI.

### Come funziona:

- Viene integrato **direttamente nei modelli di generazione** (Imagen, Gemini).
- Il watermark è **invisibile** e **codificato nei pixel** usando trasformazioni robuste.
- Può essere rilevato anche **dopo compressione, crop o filtri leggeri**.
- **Solo Google** dispone (per ora) del detector compatibile.

📽️ Video ufficiale SynthID:

<iframe width="560" height="315" src="https://www.youtube.com/embed/9btDaOcfIMY" title="SynthID - DeepMind" frameborder="0" allowfullscreen></iframe>

### 👁️ Un doppio strato di trasparenza

Google applica **due livelli di segnalazione**:

1. **SynthID** invisibile nei pixel  
2. **Logo “AI” visibile** nell’angolo dell’immagine (es. da Gemini), utile per riconoscimento umano immediato

## 📌 Conclusioni

- I **metadati C2PA** rappresentano un'ottima base per la trasparenza digitale, ma sono **facilmente rimovibili** da editor non compatibili.
- I **watermark invisibili** offrono maggiore **resistenza tecnica**, ma richiedono strumenti specializzati per la lettura.
- La **combinazione delle due tecniche** (dove possibile) è oggi la soluzione più promettente per garantire **origine e integrità** delle immagini AI.

## 🔗 Link utili

- [https://contentcredentials.org/verify](https://contentcredentials.org/verify)
- [https://c2pa.org](https://c2pa.org)
- [https://deepmind.google/technologies/synthid](https://deepmind.google/technologies/synthid)
- [https://opensource.contentauthenticity.org](https://opensource.contentauthenticity.org)
- [Wikipedia: Steganografia](https://en.wikipedia.org/wiki/Steganography)
- [Wikipedia: Discrete Cosine Transform](https://en.wikipedia.org/wiki/Discrete_cosine_transform)
- [Article on Wavelet Watermarking](https://www.researchgate.net/publication/24346540_Wavelet-based_digital_image_watermarking)

