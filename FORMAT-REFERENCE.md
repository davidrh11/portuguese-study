# Portuguese Study Guide — Format Reference

This is the canonical spec for every interactive HTML guide in `davidrh11.github.io/portuguese-study`. When building or editing a guide, this document — not chat history — is the source of truth for patterns, CSS, and JS.

**Live code references in the project files:** `restaurante.html` and `aula-15-maio.html` are the most complete, current examples of every pattern below. `index.html` is the canonical index structure. When in doubt, read those files directly rather than reconstructing from this doc — this doc can drift, the code on GitHub is reality.

---

## 1. Theme & Page Skeleton

### CSS variables (light theme — all guides)
```css
:root {
  --bg: #f9f6f0; --surface: #ffffff; --border: #e2ddd6;
  --green: #2d7a50; --red: #a03428; --text: #1a1a1a; --muted: #6b6560;
  --highlight: #eef6f1;
  --mono: 'IBM Plex Mono', monospace; --sans: 'IBM Plex Sans', sans-serif;
}
```
`--green` is the primary accent (Portuguese flag green). `--red` is secondary/error accent only — never used as primary.

### Required `<head>` elements
- `<html lang="pt" translate="no">` — the `translate="no"` is mandatory, prevents Chrome's auto-translate bug from flipping PT text to English mid-session.
- No-cache meta tags (mandatory on every file):
  ```html
  <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
  <meta http-equiv="Pragma" content="no-cache">
  <meta http-equiv="Expires" content="0">
  ```
- Font import: IBM Plex Mono (400,600) + IBM Plex Sans (300,400,600)

### Header
Every guide has a small inline SVG Portuguese flag (green/red rectangles + armillary sphere circles) in the top-left of the header, next to the title/subtitle block. Copy verbatim from `restaurante.html` or `aula-15-maio.html` — do not regenerate.

### Tabs
Standard tab set, in this order: **Tabela · Flashcards · Quiz · Desafio**. Some guides add extra tabs (e.g. aula-15-maio has a "Futuro" tab between Tabela and Flashcards) — extra tabs go *between* Tabela and Flashcards, never after Desafio.

```javascript
function switchTab(id, btn) {
  document.querySelectorAll('.view').forEach(v => v.classList.remove('active'));
  document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
  document.getElementById(id).classList.add('active');
  btn.classList.add('active');
  if (id === 'flashcards') vfcRender();
  if (id === 'quiz') startQuiz();
  if (id === 'desafio') initDesafio();
}
```

---

## 2. Audio / TTS Infrastructure

### Worker URL — current production value
```javascript
const WORKER_URL = "https://portuguese-tts-speechify.drh-dave.workers.dev";
```
This is **Speechify** (voice: **Agueda**, female, pt-PT, model `simba-multilingual`), as of the switch from Azure. The old Azure URL was `https://portuguese-tts.drh-dave.workers.dev` — if you see that string in a file, it's stale and needs updating.

### Standard speak function (exact — copy verbatim)
```javascript
const audioCache = {};
async function speak(text) {
  if (!text) return;
  if (audioCache[text]) { audioCache[text].currentTime = 0; audioCache[text].play(); return; }
  try {
    const res = await fetch(WORKER_URL, { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({text}) });
    if (!res.ok) throw new Error();
    const blob = await res.blob();
    const url = URL.createObjectURL(blob);
    const audio = new Audio(url); audioCache[text] = audio; audio.play();
  } catch(e) {}
}
```
This must sit **inside** the `<script>` tag, near the top, before any function that calls `speak()`. (A past bug: an edit accidentally stripped the `<script>` tag + this whole block from `possessivos.html`, leaving the rest of the JS as dead text in the page — always verify `<script>` is present and `speak` is fully defined after edits.)

### Text rules for TTS
- **Never use slash notation** (`obrigado/a`, `ótimo/a`) — breaks pronunciation. Dave is male: always write the full masculine form (`obrigado`), with a note that the feminine exists if relevant. If both forms need drilling, create two separate complete entries.
- **Escape both quote types in onclick attributes.** If a PT sentence contains literal double quotes (e.g. `"Algo mais?" "Não, é só."`), they will break out of a double-quoted `onclick="..."` attribute. Always chain `.replace(/'/g,"\\'").replace(/"/g,'&quot;')` when interpolating user-facing text into an `onclick` attribute.
- **Don't pass a `language_code` param** to the TTS backend — unsupported, causes errors (legacy ElevenLabs note, may not apply to Speechify but keep the discipline of not over-specifying).
- **Filenames with spaces** (for `MY_RECORDINGS`, audio files in `/audio/`) — use underscores/hyphens, never raw spaces.

### MY_RECORDINGS (Dave's own voice clips)
Some guides route specific phrases to Dave's own `.m4a` recordings (stored in `/audio/` on GitHub) instead of TTS, via a `MY_RECORDINGS` map checked before the TTS fetch. Ensure `MY_RECORDINGS` is defined *before* any function that reads it (past bug: execution-order issue caused `MY_RECORDINGS` to be undefined at call time).

---

## 3. Tabela Mode

Standard table for vocab/phrases:
```css
table { width: 100%; border-collapse: collapse; margin-bottom: 1rem; max-width: 680px; }
th { font-family: var(--mono); font-size: 0.65rem; text-transform: uppercase; letter-spacing: 0.08em; color: var(--muted); text-align: left; padding: 0.5rem 1rem; border-bottom: 1px solid var(--border); }
td { padding: 0.7rem 1rem; border-bottom: 1px solid var(--border); vertical-align: top; }
tr:last-child td { border-bottom: none; }
td.pt-phrase { font-family: var(--mono); font-size: 0.88rem; color: var(--green); font-weight: 600; cursor: pointer; width: 50%; }
td.pt-phrase:hover { text-decoration: underline; text-decoration-color: var(--green); text-underline-offset: 3px; }
td.en-phrase { color: var(--text); font-size: 0.85rem; width: 30%; }
td.note-cell { color: var(--muted); font-size: 0.8rem; width: 20%; }
td.note-cell[onclick] { cursor: pointer; }
td.note-cell[onclick]:hover { color: var(--green); }
tbody tr:hover { background: #f4f0ea; }
```

**Every PT cell with speakable content gets `onclick="speak('...')"`.** This includes the `note-cell` column when it contains a full example sentence (not just a grammar note) — add the onclick + `style="cursor:pointer"` + `title="🔊 ouvir"`. Pure grammar notes (e.g. "atrasado = late (masc.)") stay plain text, no audio.

### Vocab grid (alternative to table, for word lists)
```css
.vocab-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(180px, 1fr)); gap: 0.6rem; margin-bottom: 1.5rem; max-width: 680px; }
.vocab-card { background: var(--surface); border: 1px solid var(--border); border-radius: 6px; padding: 0.8rem 1rem; cursor: pointer; transition: border-color 0.15s; }
.vocab-card:hover { border-color: var(--green); }
.vocab-pt { font-family: var(--mono); font-size: 0.88rem; color: var(--green); font-weight: 600; }
.vocab-en { font-size: 0.78rem; color: var(--muted); margin-top: 0.2rem; }
```
Each card: `onclick="speak('...')"`, contains `.vocab-pt` + `.vocab-en` divs.

### Note blocks (grammar explanations)
```css
.note-block { background: var(--highlight); border-left: 3px solid var(--green); padding: 0.9rem 1.1rem; border-radius: 0 6px 6px 0; margin: 1rem 0; }
.note-block .note-label { font-family: var(--mono); font-size: 0.62rem; text-transform: uppercase; letter-spacing: 0.1em; color: var(--green); margin-bottom: 0.35rem; }
.note-block p { font-size: 0.87rem; line-height: 1.7; }
.note-block .pt { font-family: var(--mono); color: var(--green); font-weight: 600; }
.note-block .en { color: var(--muted); font-style: italic; }
```

### Section titles
```css
.section-title { font-family: var(--mono); font-size: 0.68rem; text-transform: uppercase; letter-spacing: 0.12em; color: var(--green); margin: 2rem 0 0.75rem; border-left: 3px solid var(--green); padding-left: 0.6rem; }
.section-title:first-child { margin-top: 0; }
```

---

## 4. Flashcards — "vfc" Pattern (exact, do not reinvent)

### CSS
```css
.flashcard-wrap { max-width: 480px; margin: 0 auto; }
.vfc-topbar { display: flex; align-items: center; justify-content: space-between; margin-bottom: 0.75rem; }
.vfc-counter { font-family: var(--mono); font-size: 0.8rem; color: var(--muted); }
.vfc-controls { display: flex; gap: 0.5rem; }
.vfc-ctrl { font-family: var(--mono); font-size: 0.72rem; background: var(--surface); border: 1px solid var(--border); color: var(--muted); padding: 0.35rem 0.85rem; border-radius: 4px; cursor: pointer; transition: all 0.15s; }
.vfc-ctrl:hover { border-color: var(--green); color: var(--green); }
.vfc-progress { height: 2px; background: var(--border); border-radius: 1px; margin-bottom: 1.25rem; overflow: hidden; }
.vfc-progress-fill { height: 100%; background: var(--green); transition: width 0.3s; }
.vfc-card { background: var(--surface); border: 1px solid var(--border); border-radius: 8px; min-height: 300px; display: flex; flex-direction: column; align-items: center; padding: 1.5rem 2rem; text-align: center; }
.vfc-hints { font-family: var(--mono); font-size: 0.6rem; color: var(--muted); letter-spacing: 0.06em; margin-bottom: 1.25rem; }
.vfc-cat { font-family: var(--mono); font-size: 0.62rem; text-transform: uppercase; letter-spacing: 0.1em; color: var(--green); align-self: flex-start; margin-bottom: 0.75rem; }
.vfc-en { font-size: 1rem; color: var(--text); font-weight: 300; line-height: 1.5; margin-bottom: 1.1rem; }
.vfc-divider { width: 2rem; height: 1px; background: var(--border); margin-bottom: 1.1rem; }
.vfc-pt { font-family: var(--mono); font-size: 1.3rem; color: var(--green); font-weight: 700; filter: blur(7px); transition: filter 0.3s; cursor: pointer; margin-bottom: 1.1rem; background: var(--border); border-radius: 4px; padding: 0.15rem 0.6rem; display: inline-block; line-height: 1.5; }
.vfc-pt.revealed { filter: blur(0); background: transparent; }
.vfc-speak { background: var(--bg); border: 1px solid var(--border); color: var(--green); padding: 0.35rem 1.2rem; border-radius: 99px; cursor: pointer; transition: all 0.2s; font-size: 1rem; margin-bottom: 1rem; }
.vfc-speak:hover, .vfc-speak.playing { background: var(--green); color: #fff; border-color: var(--green); }
.vfc-ex-pt { font-style: italic; font-size: 0.82rem; color: var(--text); filter: blur(5px); transition: filter 0.3s; cursor: pointer; margin-bottom: 0.2rem; background: var(--border); border-radius: 3px; padding: 0.05rem 0.3rem; display: inline-block; }
.vfc-ex-pt.revealed { filter: blur(0); background: transparent; }
.vfc-ex-en { font-size: 0.75rem; color: var(--muted); filter: blur(5px); transition: filter 0.3s; background: var(--border); border-radius: 3px; padding: 0.05rem 0.3rem; display: inline-block; margin-bottom: 1rem; cursor: pointer; }
.vfc-ex-en.revealed { filter: blur(0); background: transparent; }
.vfc-nav { display: flex; gap: 1rem; margin-top: auto; }
.vfc-nav-btn { font-family: var(--mono); font-size: 0.82rem; background: var(--surface); border: 1px solid var(--border); color: var(--text); padding: 0.55rem 1.25rem; border-radius: 4px; cursor: pointer; transition: all 0.15s; }
.vfc-nav-btn:hover { border-color: var(--green); color: var(--green); }
```

### HTML
```html
<div id="flashcards" class="view">
<div class="flashcard-wrap">
  <div class="vfc-topbar">
    <span class="vfc-counter" id="vfcCounter">1 / 1</span>
    <div class="vfc-controls">
      <button class="vfc-ctrl" onclick="vfcShuffle()">⇄ Baralhar</button>
      <button class="vfc-ctrl" onclick="vfcReset()">↺ Reiniciar</button>
    </div>
  </div>
  <div class="vfc-progress"><div class="vfc-progress-fill" id="vfcFill"></div></div>
  <div class="vfc-card">
    <div class="vfc-hints" id="vfcHints">clique na frase para revelar · clique no exemplo para ouvir</div>
    <div class="vfc-cat" id="vfcCat"></div>
    <div class="vfc-en" id="vfcEn"></div>
    <div class="vfc-divider"></div>
    <div class="vfc-pt" id="vfcPt" onclick="vfcReveal()"></div>
    <button class="vfc-speak" id="vfcSpeak" onclick="vfcSpeakAnswer()">🔊</button>
    <div class="vfc-ex-pt" id="vfcExPt" onclick="vfcSpeakEx()"></div>
    <div class="vfc-ex-en" id="vfcExEn"></div>
    <div class="vfc-nav">
      <button class="vfc-nav-btn" onclick="vfcMove(-1)">← Anterior</button>
      <button class="vfc-nav-btn" onclick="vfcMove(1)">Próximo →</button>
    </div>
  </div>
</div>
</div>
```

### JS
```javascript
let vfcDeck = [...CARDS], vfcIdx = 0, vfcRevealed = false;

function vfcRender() {
  const c = vfcDeck[vfcIdx];
  vfcRevealed = false;
  document.getElementById('vfcCounter').textContent = `${vfcIdx+1} / ${vfcDeck.length}`;
  document.getElementById('vfcFill').style.width = `${((vfcIdx+1)/vfcDeck.length)*100}%`;
  document.getElementById('vfcCat').textContent = c.cat;
  document.getElementById('vfcEn').textContent = c.en;
  const pt = document.getElementById('vfcPt');
  pt.textContent = c.pt; pt.className = 'vfc-pt';
  const exPt = document.getElementById('vfcExPt');
  exPt.textContent = c.note || ''; exPt.className = 'vfc-ex-pt';
  const exEn = document.getElementById('vfcExEn');
  exEn.textContent = c.en_note || ''; exEn.className = 'vfc-ex-en';
  exEn.onclick = () => exEn.classList.add('revealed');
  document.getElementById('vfcHints').textContent = 'clique na frase para revelar · clique no exemplo para ouvir';
  document.getElementById('vfcSpeak').classList.remove('playing');
}

function vfcReveal() {
  if (vfcRevealed) return; vfcRevealed = true;
  document.getElementById('vfcPt').classList.add('revealed');
  document.getElementById('vfcExPt').classList.add('revealed');
  document.getElementById('vfcHints').textContent = '';
}
function vfcSpeakAnswer() { speak(vfcDeck[vfcIdx].pt, document.getElementById('vfcSpeak')); }
function vfcSpeakEx() { vfcReveal(); speak(vfcDeck[vfcIdx].note || vfcDeck[vfcIdx].pt, document.getElementById('vfcExPt')); }
function vfcMove(dir) { vfcIdx = (vfcIdx+dir+vfcDeck.length)%vfcDeck.length; vfcRender(); }
function vfcShuffle() { vfcDeck = [...CARDS].sort(()=>Math.random()-0.5); vfcIdx=0; vfcRender(); }
function vfcReset() { vfcDeck = [...CARDS]; vfcIdx=0; vfcRender(); }
vfcRender();
```

### CARDS data shape (required fields)
```javascript
{ cat: 'CategoryName', pt: 'Portuguese text', en: 'English meaning',
  note: 'Full PT example sentence', en_note: 'English translation of example' }
```
- `cat` — drives the small label top-left of the card
- `pt` — the blurred answer
- `en` — the prompt shown unblurred
- `note` — blurred example sentence (clicking it reveals + speaks)
- `en_note` — blurred English translation of the example (clicking only reveals, no audio)

---

## 5. Quiz Mode

```css
.quiz-wrap { max-width: 520px; margin: 0 auto; }
.quiz-score { font-family: var(--mono); font-size: 0.75rem; color: var(--muted); margin-bottom: 1.5rem; }
.quiz-q { font-family: var(--mono); font-size: 1rem; color: var(--green); text-align: center; margin-bottom: 1.25rem; min-height: 60px; line-height: 1.5; }
.quiz-prompt { font-size: 0.82rem; color: var(--muted); text-align: center; margin-bottom: 0.75rem; }
.quiz-opts { display: grid; grid-template-columns: 1fr 1fr; gap: 0.65rem; margin-bottom: 1rem; }
.quiz-opt { background: var(--surface); border: 1px solid var(--border); border-radius: 6px; padding: 0.8rem; cursor: pointer; font-size: 0.82rem; color: var(--text); transition: all 0.15s; text-align: center; }
.quiz-opt:hover:not(:disabled) { border-color: var(--green); }
.quiz-opt.correct { border-color: var(--green); background: var(--highlight); color: var(--green); }
.quiz-opt.wrong { border-color: var(--red); background: #fdf0ef; color: var(--red); }
.quiz-opt:disabled { cursor: default; }
.quiz-feedback { text-align: center; font-family: var(--mono); font-size: 0.82rem; min-height: 1.4rem; margin-bottom: 1rem; }
.quiz-feedback.ok { color: var(--green); }
.quiz-feedback.bad { color: var(--red); }
.quiz-next { display: block; margin: 0 auto; font-family: var(--mono); font-size: 0.85rem; padding: 0.6rem 2rem; background: var(--green); color: #fff; border: none; border-radius: 4px; cursor: pointer; font-weight: 600; }
.quiz-next:disabled { opacity: 0.3; cursor: default; }
```

JS: `QUIZ_POOL = CARDS.map(c => ({ q:c.pt, a:c.en }))`, then standard 4-option multiple choice, auto-speaks the PT question after answering (correct or wrong), score tracking, infinite reshuffle loop on completion. See `restaurante.html` for the full implementation — copy verbatim.

---

## 6. Desafio Mode (Production Drill)

This is the highest-value tab. **Critical behaviors — do not deviate:**

1. **8-second countdown timer** (not 5).
2. **When the timer hits zero: STOP and show ✓. Do NOT auto-reveal the answer.** Hint text changes to "tempo — toca para revelar". User must tap the answer to reveal it.
3. Self-report after reveal: ✓ Disse / ✗ Não disse buttons. Streak counter (🔥 N) appears at 3+.
4. Audio plays automatically when the answer is revealed (whether by tap or after timeout-then-tap).

### CSS
```css
.desafio-wrap { max-width: 520px; margin: 0 auto; }
.d-meta { display: flex; justify-content: space-between; align-items: center; font-family: var(--mono); font-size: 0.72rem; color: var(--muted); margin-bottom: 1.25rem; flex-wrap: wrap; gap: 0.5rem; }
.d-streak { color: var(--green); font-weight: 600; }
.d-card { background: var(--surface); border: 1px solid var(--border); border-radius: 10px; padding: 1.75rem 1.5rem 1.5rem; text-align: center; min-height: 280px; display: flex; flex-direction: column; align-items: center; justify-content: flex-start; }
.d-cat { font-family: var(--mono); font-size: 0.62rem; text-transform: uppercase; letter-spacing: 0.1em; color: var(--green); margin-bottom: 0.75rem; align-self: flex-start; }
.d-prompt { font-size: 1rem; color: var(--text); margin-bottom: 1.25rem; line-height: 1.6; max-width: 400px; }
.d-timer-ring { width: 52px; height: 52px; margin: 0 auto 1.25rem; position: relative; }
.d-timer-ring svg { transform: rotate(-90deg); }
.d-timer-ring circle { fill: none; stroke-width: 4; }
.d-timer-track { stroke: var(--border); }
.d-timer-fill { stroke: var(--green); stroke-dasharray: 138; stroke-dashoffset: 0; transition: stroke-dashoffset 1s linear, stroke 0.3s; stroke-linecap: round; }
.d-timer-fill.warn { stroke: #c47a00; }
.d-timer-fill.danger { stroke: var(--red); }
.d-timer-text { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; font-family: var(--mono); font-size: 0.9rem; font-weight: 600; color: var(--text); }
.d-divider { width: 2rem; height: 1px; background: var(--border); margin-bottom: 1.25rem; }
.d-answer { font-family: var(--mono); font-size: 1.2rem; font-weight: 600; color: var(--green); filter: blur(8px); background: #e8f4ee; border-radius: 6px; padding: 0.4rem 1rem; cursor: pointer; transition: filter 0.3s, background 0.3s; margin-bottom: 0.75rem; user-select: none; line-height: 1.6; }
.d-answer.revealed { filter: blur(0); background: transparent; }
.d-speak { background: var(--bg); border: 1px solid var(--border); color: var(--green); padding: 0.3rem 1rem; border-radius: 99px; cursor: pointer; font-size: 0.9rem; margin-bottom: 1.25rem; transition: all 0.2s; }
.d-speak:hover { background: var(--green); color: #fff; }
.d-note { font-size: 0.78rem; color: var(--muted); font-style: italic; margin-bottom: 1.25rem; min-height: 1.2rem; }
.d-actions { display: flex; gap: 0.75rem; }
.d-btn { font-family: var(--mono); font-size: 0.82rem; font-weight: 600; padding: 0.6rem 1.5rem; border: none; border-radius: 6px; cursor: pointer; transition: opacity 0.15s; }
.d-btn.disse { background: var(--green); color: #fff; }
.d-btn.nao { background: #fdf0ef; color: var(--red); border: 1px solid #dba09a; }
.d-btn:hover { opacity: 0.85; }
.d-hint { font-family: var(--mono); font-size: 0.62rem; color: var(--muted); margin-top: 1rem; letter-spacing: 0.05em; }

/* Waiter audio prompt — optional, only on guides with situational dialogue (e.g. restaurante) */
.d-waiter-wrap { margin-bottom: 1rem; display: flex; flex-direction: column; align-items: center; gap: 0.4rem; }
.d-waiter-label { font-family: var(--mono); font-size: 0.6rem; text-transform: uppercase; letter-spacing: 0.08em; color: var(--muted); }
.d-waiter-btn { background: var(--surface); border: 1px solid var(--green); color: var(--green); font-family: var(--mono); font-size: 0.8rem; padding: 0.45rem 1.2rem; border-radius: 99px; cursor: pointer; transition: all 0.2s; display: flex; align-items: center; gap: 0.4rem; }
.d-waiter-btn:hover, .d-waiter-btn.playing { background: var(--green); color: #fff; }
```

### HTML
```html
<div id="desafio" class="view">
<div class="desafio-wrap">
  <div class="d-meta">
    <span id="d-score-display">Score: 0 / 0</span>
    <span class="d-streak" id="d-streak-display"></span>
  </div>
  <div class="d-card">
    <span class="d-cat" id="d-cat"></span>
    <!-- Optional waiter prompt block — only if guide uses waiter_prompt field -->
    <div class="d-waiter-wrap" id="d-waiter-wrap" style="display:none">
      <span class="d-waiter-label">🎙 o empregado pergunta:</span>
      <button class="d-waiter-btn" id="d-waiter-btn" onclick="playWaiterPrompt()">🔊 <span id="d-waiter-text"></span></button>
    </div>
    <div class="d-prompt" id="d-prompt"></div>
    <div class="d-timer-ring">
      <svg width="52" height="52" viewBox="0 0 52 52">
        <circle class="d-timer-track" cx="26" cy="26" r="22"/>
        <circle class="d-timer-fill" id="d-timer-fill" cx="26" cy="26" r="22"/>
      </svg>
      <div class="d-timer-text" id="d-timer-text">8</div>
    </div>
    <div class="d-divider"></div>
    <div class="d-answer" id="d-answer" onclick="revealDesafio()"></div>
    <button class="d-speak" onclick="speakDesafio()">🔊</button>
    <div class="d-note" id="d-note"></div>
    <div class="d-actions" id="d-actions" style="display:none">
      <button class="d-btn disse" onclick="scoreDesafio(true)">✓ Disse</button>
      <button class="d-btn nao" onclick="scoreDesafio(false)">✗ Não disse</button>
    </div>
    <div class="d-hint" id="d-hint">diz em voz alta · depois clica para revelar</div>
  </div>
</div>
</div>
```

### JS (8s timer, no auto-reveal — exact)
```javascript
let dDeck=[], dIdx=0, dCorrect=0, dTotal=0, dStreak=0, dTimer=null, dSecondsLeft=8, dRevealed=false;

function initDesafio() {
  dDeck=[...DESAFIO_CARDS].sort(()=>Math.random()-0.5);
  dIdx=0; dCorrect=0; dTotal=0; dStreak=0;
  loadDesafioCard();
}

function loadDesafioCard() {
  dRevealed=false; clearInterval(dTimer); dSecondsLeft=8;
  const c=dDeck[dIdx%dDeck.length];
  document.getElementById('d-cat').textContent=c.cat;
  document.getElementById('d-prompt').textContent=c.prompt;
  document.getElementById('d-answer').textContent=c.answer;
  document.getElementById('d-answer').classList.remove('revealed');
  document.getElementById('d-note').textContent=c.note||'';
  document.getElementById('d-actions').style.display='none';
  document.getElementById('d-hint').style.display='block';
  document.getElementById('d-hint').textContent='diz em voz alta · depois clica para revelar';
  document.getElementById('d-score-display').textContent=`Score: ${dCorrect} / ${dTotal}`;
  document.getElementById('d-streak-display').textContent=dStreak>=3?`🔥 ${dStreak}`:'';

  // Waiter prompt (optional field)
  const waiterWrap = document.getElementById('d-waiter-wrap');
  const waiterBtn = document.getElementById('d-waiter-btn');
  const waiterText = document.getElementById('d-waiter-text');
  if (c.waiter_prompt) {
    waiterWrap.style.display = 'flex';
    waiterText.textContent = c.waiter_prompt;
    waiterBtn.classList.remove('playing');
    waiterBtn.dataset.prompt = c.waiter_prompt;
  } else {
    waiterWrap.style.display = 'none';
    waiterText.textContent = '';
  }

  const fill=document.getElementById('d-timer-fill'), txt=document.getElementById('d-timer-text');
  fill.style.strokeDashoffset=0; fill.className='d-timer-fill'; txt.textContent=8;
  dTimer=setInterval(()=>{
    dSecondsLeft--;
    txt.textContent=dSecondsLeft;
    fill.style.strokeDashoffset=138*(1-dSecondsLeft/8);
    if(dSecondsLeft<=2) fill.classList.add('warn');
    if(dSecondsLeft<=1){fill.classList.remove('warn');fill.classList.add('danger');}
    if(dSecondsLeft<=0){
      clearInterval(dTimer);
      txt.textContent='✓';
      fill.style.stroke='var(--border)';
      document.getElementById('d-hint').textContent='tempo — toca para revelar';
    }
  },1000);
}

function revealDesafio() {
  if(dRevealed) return; dRevealed=true; clearInterval(dTimer);
  document.getElementById('d-timer-text').textContent='✓';
  document.getElementById('d-answer').classList.add('revealed');
  document.getElementById('d-actions').style.display='flex';
  document.getElementById('d-hint').style.display='none';
  speak(dDeck[dIdx%dDeck.length].answer);
}

function speakDesafio(){speak(dDeck[dIdx%dDeck.length].answer);}

function playWaiterPrompt(){
  const btn = document.getElementById('d-waiter-btn');
  const prompt = btn.dataset.prompt;
  if (!prompt) return;
  btn.classList.add('playing');
  speak(prompt);
  setTimeout(() => btn.classList.remove('playing'), 3000);
}

function scoreDesafio(correct){
  dTotal++; if(correct){dCorrect++;dStreak++;}else{dStreak=0;}
  dIdx++; loadDesafioCard();
}
```

### DESAFIO_CARDS data shape
```javascript
{ cat: 'CategoryName', prompt: 'English instruction/situation', answer: 'Portuguese response',
  note: 'Grammar/usage note', waiter_prompt: 'Optional PT question the waiter asks first' }
```
`waiter_prompt` is optional — only used in guides with situational dialogue (currently `restaurante.html`). When present, it renders the audio button above the prompt. Mix waiter-prompt and plain-prompt cards in the same deck for variety — don't make every card a waiter prompt.

---

## 7. Index.html Structure

`SECTIONS` array drives the whole index, rendered dynamically. Sections, in order:

1. **Aulas Recentes** — rolling max of **4** items. When a 5th aula is added, the oldest retires to **Arquivo** (bottom section, `📁` icon, same `desc`/tags preserved). Newest aula goes at the *top* of this list.
2. **Gramática** — has `subsections`: Verbos-Combinações, Verbos-Conjugações, Frases & Estrutura.
3. **Vocabulário**
4. **Expressões**
5. **Áudio para Caminhar**
6. **Arquivo** — retired aulas, `📁` icon, `color: 'muted'`.

Item shape: `{ num:'🗓'|'→'|'★'|'📁', file:'name.html', title:'...', desc:'...', tags:['desafio','new'] }`. The `'new'` tag should only be on the single most-recently-added item.

`.section-label.muted { color: var(--muted2); border-left-color: var(--border); }` — required CSS for the Arquivo section label.

---

## 8. Workflow Rules

1. **Always work from a freshly uploaded file.** Never edit from `/mnt/project/` snapshots — they go stale and silently regress completed work (e.g. reverting light theme to dark, losing added sections). If a file isn't freshly uploaded in *this* conversation, ask for it before editing.
2. **One source of truth for code patterns is this doc + `restaurante.html`/`aula-15-maio.html`/`index.html` as live references** — not chat history summaries, which lose exact JS.
3. **After any edit, verify `<script>` tags and core function declarations (`speak`, `switchTab`) are intact** — str_replace edits across large blocks have previously deleted these accidentally.
4. **Caderno/photo vocab additions:** split by source. Content traceable to a specific lesson's slides/audio → that lesson's aula guide. New standalone vocab → the current/active aula guide. Add to all three layers: Tabela (table row or vocab card with onclick audio), `CARDS` (with `note`+`en_note`), `DESAFIO_CARDS`.
5. **GitHub upload workflow:** download from chat → GitHub repo `davidrh11/portuguese-study` → Add file → upload → live in ~60s.
