---
title: "Un esempio pratico con Claude Code: migliorare un progetto open source"
date: 2025-10-01
excerpt: "Case study reale: dall’analisi automatica alla validazione degli input, con test e documentazione aggiornati, su un piccolo progetto open source."
tags: [AI Coding, Claude, Open Source, DevTools, ONNX]
classes: wide
header:
  overlay_image: /assets/images/claude-code/esempio-claude-code-06.png
  overlay_filter: 0.1
---

## Introduzione: cos’è Claude Code e perché è interessante
[Claude Code](https://www.claude.com/product/claude-code) è un ambiente pensato per **scrivere e migliorare codice insieme a un’IA**.  
Non si limita a generare funzioni o script: analizza la codebase, propone modifiche, aiuta con refactoring e test, installa dipendenze e può modificare i file in autonomia.

> Nota: online esistono già numerose guide — introduttive e avanzate — su come usare Claude per programmare.  
> Questo articolo **non è una guida completa**, ma un **esempio pratico** applicato a un piccolo progetto open per mostrare **potenzialità** e **semplicità d’uso** dello strumento.

---

## Il progetto di partenza
Come esempio pratico ho usato un mio piccolo progetto open source:  
👉 [onnx-model-generator-docker](https://github.com/asoldano/onnx-model-generator-docker)

È un servizio REST Dockerizzato che converte modelli HuggingFace in formato [Open Neural Network Exchange](https://onnx.ai/) (ONNX) utilizzando la libreria [`onnxruntime-genai`](https://github.com/microsoft/onnxruntime-genai): riceve una richiesta HTTP e restituisce un modello ottimizzato pronto da scaricare.  
Il progetto funziona, ma si tratta di un semplice prototipo e c'è spazio per migliorare **qualità, sicurezza e test**.

---

## Attivare Claude Code
Ho collegato il repository e lanciato `/init`, che analizza la codebase e genera `CLAUDE.md`, una guida interna per l’IA con struttura del progetto, dipendenze, punti critici e possibili miglioramenti.

{% include figure image_path="/assets/images/claude-code/esempio-claude-code-01.png" alt="Analisi iniziale e generazione CLAUDE.md" caption="Analisi iniziale e generazione di CLAUDE.md" %}
{% include figure image_path="/assets/images/claude-code/esempio-claude-code-02.png" alt="Analisi iniziale e generazione CLAUDE.md (parte 2)" caption="Analisi iniziale e generazione di CLAUDE.md (parte 2)" %}

---

## I suggerimenti di miglioramento
Dopo l’analisi, ho chiesto a Claude Code di suggerire possibili migliorie al progetto e il tool ha prodotto una roadmap per priorità:  
- **Critico (Sicurezza & Stabilità)**: validazione parametri di input, gestione token, rate limiting…  
- **Alto (Performance & Osservabilità)**: logging avanzato, health checks, metriche…  
- **Medio/Basso**: modularizzazione del codice, OpenAPI docs, caching strategy…

{% include figure image_path="/assets/images/claude-code/esempio-claude-code-03.png" alt="Roadmap dei miglioramenti" caption="Roadmap dei miglioramenti suggerita da Claude Code" %}
{% include figure image_path="/assets/images/claude-code/esempio-claude-code-04.png" alt="Roadmap dei miglioramenti (parte 2)" caption="Roadmap dei miglioramenti suggerita da Claude Code (parte 2)" %}

Ho scelto di partire dal punto più importante: **validare i parametri di input**.

---

## Applicare la validazione degli input
Il parametro `model_name` non veniva validato. Claude Code ha proposto e implementato controlli:

- stringa obbligatoria  
- massimo 200 caratteri  
- regex per il formato (alfanumerici, punti, trattini, underscore e slash)  
- blocco path traversal (`../`)  
- blocco di slash multipli consecutivi

**Passaggi principali:**

{% include figure image_path="/assets/images/claude-code/esempio-claude-code-05.png" alt="Selezione del miglioramento critico" caption="Selezione del miglioramento critico: input validation" %}
{% include figure image_path="/assets/images/claude-code/esempio-claude-code-06.png" alt="Diff su main.py" caption="Diff su `main.py` con le nuove regole di validazione" %}

---

## Test, risoluzione problemi e verifica
L’IA ha predisposto anche i test: aggiunta di `pytest` e `pytest-cov` in `requirements.txt`, creazione di `test_validation.py` e fix di ambiente per usare directory temporanee prima dell’import di `main.py`.

{% include figure image_path="/assets/images/claude-code/esempio-claude-code-07.png" alt="Creazione test" caption="Creazione del file di test e setup / dipendenze" %}
{% include figure image_path="/assets/images/claude-code/esempio-claude-code-08.png" alt="Fix dei test" caption="Fix dei test" %}
{% include figure image_path="/assets/images/claude-code/esempio-claude-code-09.png" alt="Fix dei test (parte 2)" caption="Fix dei test (parte 2)" %}
{% include figure image_path="/assets/images/claude-code/esempio-claude-code-10.png" alt="Fix dei test (parte 3)" caption="Fix dei test (parte 3)" %}

---

## Riepilogo finale
Al termine: **validazione robusta in `main.py`**, **12 test automatici tutti verdi**, **dipendenze e documentazione aggiornate**, con comandi rapidi per eseguire i test e fare build.

{% include figure image_path="/assets/images/claude-code/esempio-claude-code-11.png" alt="CLAUDE.md aggiornato" caption="`CLAUDE.md` aggiornato con comandi di test e build" %}
{% include figure image_path="/assets/images/claude-code/esempio-claude-code-12.png" alt="Report conclusivo" caption="Report conclusivo: vettori di attacco bloccati e tutti i test passano" %}

---

## Risultato e riflessioni
In sintesi, con Claude Code ho:
1. Ottenuto un’analisi completa e una roadmap chiara.  
2. Implementato automaticamente la validazione degli input.  
3. Verificato tutto con test generati dall’IA.  
4. Aggiornato documentazione e comandi operativi.

Il progetto è ora **più robusto e sicuro**, con un effort molto inferiore rispetto a farlo a mano.

---

## Conclusioni (tempi, profili e valore formativo)
- **Programmatore esperto (con conoscenza dello stack):** probabilmente avrebbe impiegato un tempo simile (o poco più). Nel mio caso, con Claude il flusso ha richiesto **~15 minuti netti** dall’analisi all’esito positivo dei test.  
- **Programmatore esperto ma fuori dallo stack/language del progetto:** qui l’IA tende a far **risparmiare tempo**, colmando rapidamente i gap di contesto (setup, convenzioni, dipendenze, entrypoint).  
- **Sviluppatore junior:** senza guida avrebbe impiegato **più tempo** con **risultato peggiore**. In questo scenario Claude diventa soprattutto **un’occasione di apprendimento**: capire i rischi nei sorgenti (path traversal, command injection, input sanitization), vedere test di regressione e buone pratiche applicate a un caso reale.

In ogni caso, **la revisione umana del codice resta necessaria**: l’IA accelera e alza la baseline di qualità, ma la responsabilità finale del software è di chi lo mantiene.

