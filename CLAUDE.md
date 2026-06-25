# Concentra't — Documentació tècnica per a Claude

## Visió general

**Concentra't** és una aplicació web Pomodoro pensada per al personal PTGAS d'una universitat.
És un únic fitxer HTML autocontingut (`pomodoro-ptgas.html`): tot el CSS i el JavaScript van incrustats,
sense dependències externes excepte Google Fonts. No hi ha servidor ni API; totes les dades
es guarden al **localStorage** del navegador.

---

## Arquitectura

### Estructura del fitxer

```
pomodoro-ptgas.html
├── <head>
│   ├── @import Google Fonts (DM Serif Display, DM Mono, DM Sans)
│   └── <style>  ← Tot el CSS (paletes, components, responsive)
├── <body>
│   ├── <header>          ← Capçalera + nav desktop + hamburger
│   ├── #mobile-nav       ← Menú desplegable per a mòbil
│   ├── #daily-goal-bar   ← Barra d'objectiu diari (condicional)
│   ├── main#view-timer   ← Vista Cronòmetre
│   ├── main#view-topics  ← Vista Gestió de temes
│   ├── main#view-stats   ← Vista Estadístiques
│   ├── Modals (overlay)
│   │   ├── #modal-activity     ← Registre d'activitat (tema + origen + etiquetes + nota)
│   │   ├── #modal-newtopic     ← Tema ràpid des del cronòmetre
│   │   ├── #modal-origens      ← Gestió d'origens (sistema + usuari)
│   │   ├── #modal-etiquetes    ← Gestió d'etiquetes d'usuari
│   │   ├── #modal-appearance   ← Selector de paleta
│   │   └── #modal-import       ← Importació de dades JSON
│   ├── #notification     ← Toast de notificació
│   └── <script>          ← Tot el JavaScript
```

### Capes de JavaScript

| Capa | Descripció |
|------|-----------|
| **Claus i perfils** | `KEY_THEME`, `KEY_PROFILES`, `KEY_ACTIVE`; funció `k(suffix)` que genera la clau per al perfil actiu |
| **Migració** | `migrateLSTagsToOrigens()` — migra `_tags` → `_origens` al localStorage; `migrateSessionFormat()` — converteix sessions v1.x (sense `records[]`) al nou format i assegura els camps `origens`/`etiquetes` |
| **Capa de dades** | `loadTopics/saveSessions/loadOrigens/loadEtiquetes/…` — lectura i escriptura al localStorage |
| **Origens de sistema** | `SYSTEM_ORIGENS` — constant (no al localStorage); origens "Whats" i "e-Admin" sempre disponibles |
| **Estat del cronòmetre** | Variables globals: `timerState`, `phase`, `secondsLeft`, `pomodorosThisSession`, etc. |
| **UI / Render** | Funcions `render*()` que reescriuen fragments del DOM |
| **Lògica de negoci** | `startTimer`, `tick`, `phaseComplete`, `finishTimer`, etc. |
| **Registre d'activitat** | `openActivityModal`, `saveActivityRecord`, `upsertDraftSession` — flux de registre per pomodoro |
| **Exportació/importació** | `exportCSV`, `exportJSON`, `confirmImport` |
| **Init** | IIFE que executa migració, carrega el perfil i inicialitza la UI |

---

## Model de dades (localStorage)

### Claus globals (no depenen del perfil)
| Clau | Valor |
|------|-------|
| `concentrat_theme` | ID de la paleta activa (`"default"`, `"dark"`, …) |
| `concentrat_profiles` | JSON array de noms de perfil, p.ex. `["default","treball"]` |
| `concentrat_active` | Nom del perfil actiu, p.ex. `"default"` |

### Claus per perfil (`concentrat_{PERFIL}_{suffix}`)
| Suffix | Tipus | Descripció |
|--------|-------|-----------|
| `topics` | `Topic[]` | Llista de temes de treball |
| `sessions` | `Session[]` | Historial de sessions (noves primer) |
| `origens` | `Origen[]` | Origens d'usuari (≠ origens de sistema) |
| `etiquetes` | `Etiqueta[]` | Etiquetes d'usuari (sistema independent) |
| `settings` | `Settings` | Configuració de temps i objectiu diari |

> **Nota de migració**: la clau `_tags` (v2.2.x) es migra automàticament a `_origens`
> la primera vegada que s'executa `bootApp()` via `migrateLSTagsToOrigens()`.

### Estructures de dades

```typescript
interface Topic {
  id: string;       // timestamp string
  name: string;
  color: string;    // hex color
  // NOTA: els camps sessions i minutes ja no s'actualitzen ni es mostren;
  // els comptadors es calculen dinàmicament des de Session.records[]
}

interface Session {
  id: number;           // Date.now() en crear la sessió
  date: string;         // ISO 8601 — inici de sessió
  endDate: string;      // ISO 8601 — fi de sessió
  pomodoros: number;    // pomodoros completats
  breakCount: number;
  totalMinutes: number; // suma de tots els Record.minutes
  records: Record[];    // registres d'activitat (un per pomodoro registrat)
}

interface Record {
  id: string;           // timestamp string
  sessionId: string;
  topicId: string;
  topicName: string;
  origens: string[];    // array d'IDs d'Origen (sistema o usuari)
  etiquetes: string[];  // array d'IDs d'Etiqueta d'usuari
  note: string;         // màx. 200 car.
  startTime: string;    // ISO 8601
  endTime: string;      // ISO 8601
  minutes: number;      // minuts de treball d'aquest registre
}

// Origens d'usuari (es guarden a localStorage)
interface Origen {
  id: string;    // timestamp string
  name: string;
  color: string; // hex color
}

// Origens de sistema (constants al JS, NO al localStorage)
// SYSTEM_ORIGENS = [
//   { id: 'sys_whats',  name: 'Whats',   color: '#25D366', system: true },
//   { id: 'sys_eadmin', name: 'e-Admin', color: '#003B7A', system: true },
// ]

// Etiquetes d'usuari (es guarden a localStorage, sistema independent d'Origen)
interface Etiqueta {
  id: string;    // timestamp string
  name: string;
  color: string; // hex color (input[type=color] lliure)
}

interface Settings {
  work: number;       // minuts de treball (defecte: 25)
  short: number;      // pausa curta (defecte: 5)
  long: number;       // pausa llarga (defecte: 15)
  longAfter: number;  // pomodoros fins pausa llarga (defecte: 4)
  dailyGoal: number;  // objectiu diari en pomodoros (0 = desactivat)
}
```

**Font de veritat única**: totes les estadístiques (gràfics, comptadors de temes, mapa de calor)
es calculen sempre des de `Session.records[]`. Els camps `Topic.sessions` i `Topic.minutes`
existeixen per compatibilitat amb backups antics però **no s'usen per a la visualització**.

Sessions antigues amb camp `records[].tags` (v2.2.x) es migren automàticament a `records[].origens`
en la funció `migrateSessionFormat()`. El camp `records[].etiquetes` s'inicialitza a `[]` si no existeix.

---

## Funcionalitats implementades

### 1. Cronòmetre Pomodoro
- Cicle: treball → modal de registre → pausa automàtica → treball automàtic → …
- Estats: `idle` | `running` | `paused`
- Barra de progrés animada al capdamunt de la targeta
- Colors d'alerta: taronja (25% restant) i vermell parpellejant (15%)
- Punts de pomodoro (dots) que reflecteixen el cicle actual
- **So de campana** (Web Audio API, harmònics reals) en cada fi de pomodoro
- **Timestamps absoluts** (`phaseEndTime`): `tick()` calcula `secondsLeft` per diferència amb `Date.now()` en lloc de decrementar. Resisteix bloqueigs de pantalla, suspensions del sistema i canvis de pestanya. La guarda `phaseCompleting` evita que `phaseComplete()` s'execute dues vegades per al mateix final de fase.

### 2. Flux de registre d'activitat
1. L'usuari inicia el cronòmetre
2. Quan el pomodoro arriba a 0: **so de campana** + **modal** (espera activa)
3. L'usuari selecciona tema, origen, etiquetes, afegeix nota i prem **"Desa el registre"**
   - El registre es guarda **immediatament** a `localStorage` via `upsertDraftSession()`
   - La pausa (curta o llarga) s'inicia automàticament
4. Si l'usuari prem **"Descarta"**: el temps no es registra, però la pausa sí que s'inicia
5. Quan la pausa acaba: el treball s'arranca automàticament (sense modal)
6. El cicle es repeteix

### 2b. Fi manual ("Acabar sessió")
- Disponible en qualsevol moment durant treball o pausa
- Obre el modal de registre si hi ha temps sense registrar
- No inicia cap pausa automàtica

### 2c. Auto-save (protecció contra pèrdua de dades)
- Cada vegada que l'usuari desa un registre al modal, `upsertDraftSession()` persisteix
  la sessió activa (amb tots els registres fins ara) directament a `localStorage`
- Si el navegador es tanca inesperadament, els registres ja confirmats apareixeran
  a les estadístiques en reobrir l'aplicació
- `upsertDraftSession()` fa un `unshift` (nova sessió) o actualitza la sessió existent
  pel seu `id` si ja existia (flux de múltiples pomodoros en una mateixa sessió)

### 3. Gestió de temes
- CRUD de temes amb color personalitzable (paleta de 8 colors)
- Comptadors de sessions i minuts calculats des dels registres reals (sempre consistents)

### 4. Sistema d'Origen
Dos tipus d'origens, gestionats per separat:

**Origens de sistema** (constants al JS, no editables des de la UI):
- "Whats" (`sys_whats`, #25D366)
- "e-Admin" (`sys_eadmin`, #003B7A)
- Es mostren amb el badge 🔒 sistema a la llista i sense botons d'editar/eliminar
- Sempre disponibles al modal de registre d'activitat

**Origens d'usuari** (emmagatzemats a `concentrat_{PERFIL}_origens`):
- CRUD amb color lliure
- S'assignen per registre en el modal d'activitat
- Es mostren com a chips (`origen-chip`, fons semitransparent) a la taula d'estadístiques
- Filtre per origen a les estadístiques
- Gràfic de barres "Minuts per origen"

### 5. Sistema d'Etiquetes
Sistema independent d'Origen, emmagatzemat a `concentrat_{PERFIL}_etiquetes`:
- CRUD d'etiquetes d'usuari amb color lliure (cap etiqueta de sistema)
- S'assignen per registre en el modal d'activitat (secció separada per un divisor)
- Es mostren com a chips (`etiqueta-chip`, vora puntejada) a continuació dels chips d'origen
- Filtre per etiqueta a les estadístiques (aplicable simultàniament amb el filtre d'origen)
- Gràfic de barres "Minuts per etiqueta"

### 6. Estadístiques
- Filtres: període (setmana / mes / setmana passada / tot) + tema + origen + etiqueta
- Tots els filtres són acumulables (intersecció)
- Mètriques: sessions, pomodoros, minuts totals, durada mitja
- Gràfic de barres: minuts per tema (WCAG 2.2 compliant en tots els temes)
- Gràfic de barres: minuts per origen (inclou origens de sistema)
- Gràfic de barres: minuts per etiqueta
- Mapa de calor setmanal (últimes 8 setmanes): columna de noms separada, dia d'avui destacat
- Taula de sessions recents (fins a 60) amb chips d'origen + etiqueta, tema i nota — **ordenada per `startTime` descendent** en el moment de renderitzar (sense modificar l'array al localStorage)

### 7. Exportació / importació
- **CSV**: sessions amb tema, minuts, origen, etiquetes i nota (columnes separades)
- **JSON** (versió 4): backup complet (sessions, temes, origens, etiquetes, configuració)
  - Noms de fitxer: `concentrat-{perfil}-registres-YYYY-MM-DD.csv` i `concentrat-{perfil}-backup-YYYY-MM-DD.json`
- **Importació JSON**:
  - Compatible amb backups v3 (camp `tags` → s'importa com `origens`)
  - Mode **Fusió**: afegeix ítems amb IDs nous, conserva els existents
  - Mode **Substitució**: sobreescriu totes les dades del perfil actiu
  - Validació: el fitxer ha de tenir `sessions[]` amb camps `id` i `date`

### 8. Paletes de colors (Aparença)
| ID | Nom | Descripció |
|----|-----|-----------|
| `default` | Default | Tons càlids beix/verd |
| `dark` | Fosc | Mode fosc, fons obscur |
| `contrast` | Contrast alt | Blanc/negre, WCAG |
| `university` | Universitari | Blau institucional |
| `pastel` | Pastel modern | Lila suau, menta i coral |

Implementat via **CSS custom properties** en `[data-theme]`. La selecció es guarda a `concentrat_theme`.

### 9. Perfils de dades
- Permeten tenir múltiples namespaces de dades al mateix navegador
- Útil per a: usuaris compartits, entorns (feina/personal), proves
- Operacions: crear, canviar i eliminar perfils
- Perfil `default` no es pot eliminar
- Migració automàtica de dades de la versió 1.x (`ptgas_*` → `concentrat_default_*`)

### 10. Menú hamburguesa
- Activat per sota de 768px d'amplada
- Slide-down animat via `max-height`
- Es tanca en clicar fora o en seleccionar una opció

### 11. Objectiu diari
- Camp configurable (0 = desactivat)
- Barra de progrés sota la capçalera, visible quan l'objectiu > 0
- Actualitzada en guardar cada sessió

### 12. Notificacions natives
- Usa la Notification API del navegador
- Demana permís al primer inici de sessió
- Envia notificació en cada fi de fase (pomodoro o pausa)

### 13. Recordatoris de moviment
- 10 suggeriments aleatoris d'estiraments/passejos
- Es mostren en finalitzar cada pomodoro
- Desapareixen amb "✓ Fet!" o en iniciar la nova fase

---

## Guia d'estils CSS

- **Tipografies**: `DM Serif Display` (títols), `DM Mono` (nombres/codi), `DM Sans` (cos)
- **Variables**: totes les referències de color usen `var(--...)` del tema actiu
- **Border-radius**: `--radius: 12px` per a targetes; 8px per a botons i inputs; 20px per a la targeta del cronòmetre
- **Ombres**: `var(--shadow)` (varia per tema)
- **Animacions**: `modalIn` (escala+translació), `slideUp` (recordatoris), `blink` (punts), `pulse-red` (temps crític)
- **Breakpoints**: 768px (hamburger), 600px (ajustos de padding/mida)
- **Contrast WCAG 2.2**: les barres de gràfics usen opacitat `BB` (73%) en mode fosc i `30` (19%) en la resta per complir el criteri 1.4.11 Non-text Contrast
- **Chips d'origen**: `.origen-chip` — fons semitransparent (`color22`)
- **Chips d'etiqueta**: `.etiqueta-chip` — vora puntejada (`border: 1.5px dashed currentColor`), fons transparent; diferenciació visual clara

---

## Convenció de noms de funcions

| Prefix/Sufix | Significat |
|---|---|
| `render*()` | Reescriu un fragment del DOM |
| `load*()`   | Llegeix del localStorage i retorna l'objecte |
| `save*()`   | Desa al localStorage |
| `update*()` | Actualitza un element existent (no reescriu tot) |
| `upsert*()` | Insereix o actualitza al localStorage (crea si no existeix) |
| `*Timer()`  | Controla el cicle del cronòmetre |
| `*Modal()`  | Obre o tanca un overlay |
| `*Profile()`| Gestiona perfils de dades |

### Exemples actuals de les funcions de dades

| Funció | Descripció |
|--------|-----------|
| `loadTopics()` / `saveTopics(t)` | Temes de treball |
| `loadSessions()` / `saveSessions(s)` | Sessions (aplica `migrateSessionFormat`) |
| `loadOrigens()` / `saveOrigens(t)` | Origens d'usuari |
| `loadEtiquetes()` / `saveEtiquetes(t)` | Etiquetes d'usuari |
| `loadSettings()` / `saveSettings(s)` | Configuració |
| `renderOrigensList()` | Llista origens (sistema + usuari) al modal |
| `renderEtiquetesList()` | Llista etiquetes al modal |
| `renderStatsOrigenFilter()` | Omple el selector `#filter-origen` |
| `renderStatsEtiquetaFilter()` | Omple el selector `#filter-etiqueta` |
| `renderOrigenChart(records)` | Gràfic de barres per origen |
| `renderEtiquetaChart(records)` | Gràfic de barres per etiqueta |
| `addOrigen()` / `deleteOrigen(id)` | CRUD origens d'usuari |
| `addEtiqueta()` / `deleteEtiqueta(id)` | CRUD etiquetes d'usuari |

---

## Migracions implementades

| Funció | Quan s'executa | Descripció |
|--------|---------------|-----------|
| `migrateLSTagsToOrigens()` | Al `bootApp()` | Mou la clau `concentrat_{P}_tags` a `concentrat_{P}_origens` per a tots els perfils (executa una sola vegada gràcies a la comprovació `null`) |
| `migrateSessionFormat(s)` | Al `loadSessions()` (en cada càrrega) | Converteix sessions v1.x (sense `records[]`); copia `r.tags → r.origens` si `r.origens` no existeix; inicialitza `r.etiquetes = []` si no existeix |

---

## Restriccions tècniques

- **Un únic fitxer HTML** (sense build step, sense bundler)
- **Cap framework** (vanilla JS/CSS)
- **Sense backend**: tot al localStorage
- **Google Fonts** és l'única dependència externa (CDN)
- Compatible amb **Chrome, Firefox i Edge** moderns
- Comentaris en **català**

---

## Decisions de disseny rellevants

- **Font de veritat única per als comptadors de temes**: `renderTopics()` suma des de
  `Session.records[]` en lloc d'usar `Topic.sessions`/`Topic.minutes`. Això elimina
  divergències per importació i errors de coma flotant acumulats.
- **Auto-save via `upsertDraftSession()`**: cada registre confirmat es persisteix immediatament.
  La sessió s'identifica per `currentSessionId` i s'actualitza in-place en sessions[].
- **Contrast heatmap en mode fosc**: text `#ffffff` en qualsevol cel·la amb dades quan el tema
  és `dark`; en temes clars, text blanc només si intensitat > 0.45.
- **Origens de sistema sense localStorage**: `SYSTEM_ORIGENS` és una constant JS. Això evita
  duplicats i permet actualitzar-los sense migracions. Quan es llegeixen `r.origens[]`, es
  busquen primer a `SYSTEM_ORIGENS` i després a `loadOrigens()`.
- **Diferenciació visual Origen vs Etiqueta**: `.origen-chip` usa fons semitransparent;
  `.etiqueta-chip` usa `border: dashed` i fons transparent. Fàcilment reconeixibles a la taula.
- **Modal d'activitat no es tanca automàticament**: el listener de clic extern exclou
  `#modal-activity` (`querySelectorAll('.modal-overlay:not(#modal-activity)')`).
  La funció `startBreak()` té una guarda addicional que comprova que el modal no sigui obert.
