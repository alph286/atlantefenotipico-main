# Ottimizzazione — Atlante Fenotipico di Comunità

Questo documento spiega **l'ottimizzazione principale** introdotta (miniature + full‑res
su richiesta) e risponde all'idea di una **"render distance"**.

---

## 1. Il problema: la VRAM, non il disco

Le 226 immagini sono `1920×960` px. Su disco pesano poco (webp compresso, ~51 MB totali),
ma **la GPU non lavora col file compresso**: quando una texture viene caricata, viene
*decompressa* in memoria video (VRAM) come griglia di pixel RGBA.

```
costo in VRAM di 1 immagine = larghezza × altezza × 4 byte (R,G,B,A)
                            = 1920 × 960 × 4
                            ≈ 7 MB
```

Caricandole **tutte** (com'era prima):

```
226 immagini × 7 MB ≈ 1,55 GB di VRAM
```

(con le mipmap che Three.js genera, anche un po' di più).

- Su **desktop** con scheda dedicata regge, ma è uno spreco enorme.
- Su **grafica integrata / portatili** può rallentare o dare artefatti.
- Su **mobile** è quasi garantito il crash: il browser perde il contesto WebGL
  (schermo nero / "Aw snap") perché la VRAM condivisa col sistema è poca.

> Regola d'oro: in 3D ciò che conta è il **numero di pixel** tenuti in memoria, non i KB del file.

---

## 2. La soluzione: due livelli (thumbnail + full‑res on demand)

L'idea è semplice: **non serve tenere in VRAM 226 immagini ad altissima risoluzione
mentre guardi la griglia da lontano.** Bastano delle miniature. Il full‑res lo carichi
solo per l'immagine che stai effettivamente ingrandendo.

### Livello 1 — Griglia = miniature

Una cartella `thumbs/` contiene una versione `480×240` di ogni immagine (`thumbs/001.jpg …`).

```
costo miniatura = 480 × 240 × 4 ≈ 0,44 MB
226 miniature   ≈ 100 MB di VRAM   ← invece di 1,55 GB
```

### Livello 2 — Full‑res solo quando ingrandisci

Quando clicchi un'immagine, viene caricato il `1920×960` originale **solo per quella**,
e viene **liberato** (`dispose()`) quando l'immagine torna nella griglia.

```
VRAM a regime ≈ 100 MB (tutte le miniature)
             + 7 MB × (1 o 2 immagini ingrandite alla volta)
```

### Come funziona nel codice (`index.html`)

| Funzione            | Ruolo |
|---------------------|-------|
| `thumbSrc(row,col)` | percorso miniatura → `thumbs/NNN.jpg` (usata dalla griglia) |
| `imgSrc(row,col)`   | percorso full‑res → `images/NNN.webp` (usata solo allo zoom) |
| `onTexDone`         | all'avvio carica la miniatura e la salva in `mesh.userData.thumbTex` |
| `loadFullRes(mesh)` | allo zoom carica il full‑res e fa lo swap `material.map = fullTex` |
| `releaseFullRes(mesh)` | al ritorno rimette la miniatura e fa `fullTex.dispose()` (libera la VRAM) |

Flusso di una sessione di zoom:

```
[griglia]            tutte le celle mostrano la miniatura (≈100 MB)
   │ click su A
   ▼
zoomTo(A) ─► loadFullRes(A)         carica images/A.webp in background
   │                                 (intanto resti sulla miniatura → niente buco nero)
   ▼  full-res pronto
mesh A.material.map = fullTex_A      A diventa nitida
   │ esci (ESC / click / scroll-out) o vai su un'altra immagine
   ▼
releaseFullRes(A)                    map = miniatura, fullTex_A.dispose() → VRAM liberata
```

Il punto chiave è il `dispose()`: senza di esso le texture full‑res si accumulerebbero
e torneremmo al problema iniziale. Liberandole, **al massimo 1–2 full‑res** vivono in VRAM
in ogni momento.

### Come si generano le miniature

Una‑tantum, con uno script PowerShell che usa il codec WebP integrato in Windows 11
(decodifica via WIC/WPF) e ricomprime in JPEG `480×240`. **Nessun build step, nessuna
dipendenza.** Va rilanciato solo se aggiungi/cambi immagini.

---

## 3. La tua idea: una "render distance"

Nei videogiochi la *render distance* significa: **disegna/carica solo ciò che è vicino**,
e ignora ciò che è lontano. Applichiamola al nostro atlante distinguendo due cose diverse
che spesso vengono confuse.

### a) "Render distance" sul **disegno** → ce l'hai già

Three.js fa **frustum culling** in automatico: i mesh **fuori dallo schermo non vengono
disegnati** (zero draw call per loro). Quindi mentre navighi, la GPU disegna solo le celle
inquadrate, non tutte e 226. Questo "non disegnare ciò che non vedo" è già attivo e gratuito.

> Nota: culling = "non lo *disegno*". Ma la texture **resta comunque in VRAM**. Sono due
> cose separate.

### b) "Render distance" sul **caricamento in memoria** → opzionale, oggi non serve

Una vera render distance applicata alla *memoria* vorrebbe dire:

```
per ogni frame:
  cella_camera = cella sotto la camera          (già calcolabile, vedi meshAtCursor)
  per ogni cella:
     se distanza(cella, cella_camera) < RAGGIO  → assicurati che la texture sia caricata
     altrimenti                                 → dispose della texture (liberala)
```

Cioè: tieni in VRAM solo le miniature attorno a dove stai guardando, e scarichi le lontane.

**Conviene farlo adesso?** No. Con le miniature siamo già a ~100 MB totali: è poco, tenerle
tutte è più semplice e non dà problemi. Una render distance sul caricamento aggiunge
complessità (logica di load/unload ogni frame, possibili "pop‑in" di immagini che si
caricano mentre ti muovi) per un guadagno che oggi non ci serve.

**Quando diventerebbe utile?**
- Se la collezione cresce molto (es. **1000–5000+ immagini**): lì anche le sole miniature
  pesano troppo e la render distance sul caricamento diventa la mossa giusta.
- Per spingere ancora di più sul **mobile** di fascia bassa.

In quel caso il punto di partenza è già pronto nel codice: la proiezione matematica
schermo→cella di `meshAtCursor` permette di sapere in O(1) qual è la cella centrale, da cui
calcolare il raggio di celle da tenere caricate.

---

## In sintesi

| | Prima | Ora |
|---|---|---|
| Texture in VRAM | 226 × full‑res | 226 miniature + 1–2 full‑res |
| VRAM | ~1,55 GB | ~100 MB |
| Mobile | crash probabile | molto più sostenibile |
| Disegno fuori schermo | già escluso (frustum culling) | invariato |
| Render distance sul caricamento | — | non necessaria oggi; utile se la collezione cresce molto |
