# Ottimizzazione — Atlante Fenotipico di Comunità

Questo documento spiega le ottimizzazioni introdotte (miniature + full‑res su richiesta,
fix mobile, pannelloni) e risponde all'idea di una **"render distance"**.

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
| `thumbSrcN(n)`      | percorso miniatura → `thumbs/NNN.jpg` (usata dalla griglia) |
| `imgSrcN(n)`        | percorso full‑res → `images/NNN.webp` (usata solo allo zoom) |
| `onTexDone`         | all'avvio carica la miniatura e la salva in `plane.userData.thumbTex` |
| `loadFullRes(card)` | allo zoom carica il full‑res e fa lo swap `material.map = fullTex` |
| `releaseFullRes(card)` | al ritorno rimette la miniatura e fa `fullTex.dispose()` (libera la VRAM) |

Flusso di una sessione di zoom:

```
[griglia]            tutte le celle mostrano la miniatura
   │ click su A
   ▼
zoomTo(A) ─► loadBackThumb(A)       carica la miniatura del retro (se non già caricata)
           ─► loadFullRes(A)         carica images/A.webp in background
   │                                 (intanto resti sulla miniatura → niente buco nero)
   ▼  full-res pronto
plane.material.map = fullTex_A       A diventa nitida
   │ esci (ESC / click / scroll-out) o vai su un'altra immagine
   ▼
releaseFullRes(A)                    map = miniatura, fullTex_A.dispose() → VRAM liberata
```

### Come si generano le miniature carte

Una‑tantum, con uno script PowerShell che usa il codec WebP integrato in Windows 11
(decodifica via WIC/WPF) e ricomprime in JPEG `480×240`. **Nessun build step, nessuna
dipendenza.** Va rilanciato solo se aggiungi/cambi immagini.

---

## 3. Fix mobile e ottimizzazioni renderer

Dopo aver misurato la VRAM reale con `nvidia-smi` è emerso che il consumo era ancora
troppo alto per dispositivi mobili. Sono stati introdotti diversi fix aggiuntivi.

### 3a. Il vero problema: i pannelloni

I pannelloni (`images/pannelloni/`) erano immagini **~7000×7000 px** caricate a piena
risoluzione all'avvio, inclusi i retri che non vengono mai mostrati (i pannelloni non
sono zoomabili né girabili).

```
dovesihannoipiedi  7077×7099 px → 191 MB VRAM × 2 facce = 383 MB
ognitanto          7086×3538 px →  96 MB VRAM × 2 facce = 192 MB
perandarevia       7112×3542 px →  96 MB VRAM × 2 facce = 192 MB
unmondopossibile   7112×3548 px →  96 MB VRAM × 2 facce = 192 MB
─────────────────────────────────────────────────────────────────
Totale pannelloni                                        ≈ 959 MB
```

**Fix:** miniature 1/8 scala per i fronti, retri non caricati.

```python
# script Python (una-tantum) per generare thumbs/pannelloni/
from PIL import Image
import os

src = 'images/pannelloni/'
dst = 'thumbs/pannelloni/'
os.makedirs(dst, exist_ok=True)
for f in os.listdir(src):
    if not f.endswith('.webp'): continue
    img = Image.open(src + f)
    img = img.resize((img.width // 8, img.height // 8), Image.LANCZOS)
    img.save(dst + f.replace('.webp', '.jpg'), 'JPEG', quality=82)
```

```
dopo:  4 fronti pannelloni × ~2 MB ≈ 7,5 MB di VRAM   ← invece di 959 MB
```

### 3b. Retri carte singole on‑demand

In griglia solo i fronti sono visibili (backface culling). I 113 retri
venivano caricati all'avvio senza essere mai mostrati finché l'utente non
fa zoom e gira la carta.

**Fix:** `loadBatch` salta i retri; `loadBackThumb(card)` li carica la prima
volta che quella carta viene aperta con `zoomTo`.

```
prima:  226 miniature caricate all'avvio  ≈ 100 MB
dopo:   113 fronti all'avvio              ≈  50 MB
        + retro caricato on-demand (≈ 0,44 MB) solo quando si apre la carta
```

### 3c. Nessuna mipmap per le miniature

Three.js genera automaticamente le mipmap (versioni 1/2, 1/4, 1/8… della
texture) che aggiungono il 33% di VRAM extra. Per le miniature non servono:
vengono sempre visualizzate a dimensione simile a quella naturale.

```javascript
tex.generateMipmaps = false;
tex.minFilter = THREE.LinearFilter;
```

Risparmio: ~33 MB (33% di 100 MB).

### 3d. Fix specifici mobile

| Fix | Dettaglio | Risparmio |
|-----|-----------|-----------|
| `antialias: !isMobile` | MSAA disabilitato su mobile | ~20–40 MB framebuffer |
| `pixelRatio` cap 1.5× su mobile | invece di 2× | ~25% framebuffer |
| `powerPreference: 'default'` su mobile | evita throttling termico | — |
| `initTexture` lazy su mobile | non blocca il main thread durante il batch‑load | — |
| Handler `webglcontextlost` | ricarica la pagina se il contesto viene perso | — |

### 3e. Risultati misurati (RTX 3060, Chrome, `nvidia-smi`)

| | Prima di tutti i fix | Dopo |
|---|---|---|
| Baseline Chrome | ~561 MiB | ~593 MiB |
| **Picco durante caricamento** | **1.785 MiB** | **832 MiB** |
| **Stabile dopo caricamento** | **~1.663 MiB** | **~800 MiB** |
| **Delta app (stabile)** | **~1.102 MiB** | **~207 MiB** |
| Riduzione | — | **−81%** |

I ~207 MB stabili si dividono in: ~58 MB texture + ~130 MB framebuffer MSAA desktop
+ ~20 MB overhead driver. Su mobile (no MSAA, DPR 1.5×) il delta stimato è ~80–100 MB.

---

## 4. La tua idea: una "render distance"

Nei videogiochi la *render distance* significa: **disegna/carica solo ciò che è vicino**,
e ignora ciò che è lontano. Applichiamola al nostro atlante distinguendo due cose diverse
che spesso vengono confuse.

### a) "Render distance" sul **disegno** → ce l'hai già

Three.js fa **frustum culling** in automatico: i mesh **fuori dallo schermo non vengono
disegnati** (zero draw call per loro). Questo "non disegnare ciò che non vedo" è già
attivo e gratuito.

> Nota: culling = "non lo *disegno*". Ma la texture **resta comunque in VRAM**. Sono due
> cose separate.

### b) "Render distance" sul **caricamento in memoria** → non necessaria oggi

Una vera render distance applicata alla *memoria* vorrebbe dire:

```
per ogni frame:
  cella_camera = cella sotto la camera          (già calcolabile, vedi meshAtCursor)
  per ogni cella:
     se distanza(cella, cella_camera) < RAGGIO  → assicurati che la texture sia caricata
     altrimenti                                 → dispose della texture (liberala)
```

**Conviene farlo adesso?** No. Con tutte le ottimizzazioni siamo a ~58 MB di texture:
è poco, tenerle tutte è più semplice e non dà problemi.

**Quando diventerebbe utile?**
- Se la collezione cresce molto (es. **1000–5000+ immagini**): lì anche le sole miniature
  pesano troppo e la render distance sul caricamento diventa la mossa giusta.
- Per spingere ancora di più sul **mobile** di fascia molto bassa.

In quel caso il punto di partenza è già pronto nel codice: la proiezione matematica
schermo→cella di `meshAtCursor` permette di sapere in O(1) qual è la cella centrale, da cui
calcolare il raggio di celle da tenere caricate.

---

## In sintesi

| | Versione originale | Ora |
|---|---|---|
| Texture in VRAM all'avvio | 226 full‑res (~1,55 GB) | 113 fronti thumb + 4 pannelloni thumb (~58 MB) |
| Retri carte | caricati all'avvio | on‑demand al primo zoom |
| Pannelloni | full‑res 7000 px (~959 MB) | thumbnail 1/8 (~7,5 MB) |
| Mipmap | generate automaticamente (+33%) | disabilitate per le miniature |
| Mobile antialias | attivo | disabilitato |
| Mobile pixel ratio | cap 2× | cap 1.5× |
| Context loss | non gestito | ricarica pagina |
| **VRAM delta app (stabile)** | **~1.102 MB** | **~207 MB (−81%)** |
| Mobile | crash probabile | stabile (~80–100 MB stimati) |
| Render distance sul caricamento | — | non necessaria; utile se collezione > 1000 immagini |
