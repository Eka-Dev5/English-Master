# 📘 README DÉVELOPPEUR — Suite *Master Quiz*

> Documentation technique ultra-précise.
> Valide pour : **English Master**, **Histoire Master**, **Géographie Master**, et tout futur **Subject Master**.
> Dernière mise à jour : mai 2025.

---

## 🗂️ Architecture générale

Chaque matière est un **dossier GitHub indépendant** contenant exactement 4 fichiers HTML autonomes.
Aucun serveur, aucune base de données, aucune dépendance externe.
Communication entre fichiers : **localStorage uniquement**.

```
quiz.html          ← Point d'entrée principal (jeu)
dashboard.html     ← Espace personnel joueur
lexique.html       ← Vocabulaire / notions interactif
index.html         ← Page d'accueil (landing page)
```

### Dépôts GitHub

| Matière | URL |
|---|---|
| English Master | `https://eka-dev5.github.io/English-Master/` |
| Histoire Master | `https://eka-dev5.github.io/Histoire/` |
| Géographie Master | `https://eka-dev5.github.io/geographie/` |

---

## 🔑 Clés localStorage — par matière

Chaque matière a ses propres clés. Elles ne doivent **jamais être partagées** entre deux matières.

| Matière | `storageKey` | `playersKey` |
|---|---|---|
| English Master | `englishMaster_v4` | `englishMaster_players` |
| Histoire Master | `histoireMaster_v1` | `histoireMaster_players` |
| Géographie Master | `geoMaster_v1` | `geoMaster_players` |

> ⚠️ `storageKey` et `playersKey` **doivent être identiques** dans `quiz.html` et `dashboard.html` d'une même matière.

### Structure d'un joueur dans le localStorage

```json
{
  "Prénom": {
    "name"          : "Prénom",
    "currentLevel"  : 3,
    "score"         : 120,
    "completed"     : [1, 2],
    "totalQuestions": 45,
    "totalCorrect"  : 38,
    "streak"        : 5,
    "lastPlayed"    : "2025-04-22T10:30:00.000Z",
    "errorHistory"  : [ ... ],
    "sessionHistory": [ ... ],
    "activeSession" : null
  }
}
```

| Propriété | Type | Rôle |
|---|---|---|
| `currentLevel` | `int` | Niveau actuel débloqué |
| `score` | `int` | Points cumulés |
| `completed` | `int[]` | Niveaux terminés avec ≥ 80% |
| `streak` | `int` | Série de bonnes réponses consécutives |
| `errorHistory` | `obj[]` | 50 dernières erreurs (pour les fiches) |
| `sessionHistory` | `obj[]` | 50 dernières sessions jouées |
| `activeSession` | `obj\|null` | Session interrompue (reprise possible) |

---

## ⚙️ SUBJECT_CONFIG — bloc de personnalisation

Dans `quiz.html`, **première chose à modifier** pour créer une nouvelle matière :

```javascript
const SUBJECT_CONFIG = {
  name         : "Histoire Master",   // Nom affiché dans l'interface
  emoji        : "🏛️",               // Emoji du cours
  lang         : "fr",               // Langue (non utilisé fonctionnellement)
  storageKey   : "histoireMaster_v1", // Clé unique localStorage
  playersKey   : "histoireMaster_players", // Clé joueurs unique
  dashboardFile: "dashboard.html",   // Nom du fichier dashboard
  lexiqueFile  : "lexique.html"      // Nom du fichier lexique
};
```

---

## 📄 quiz.html — Structure détaillée

### CSS — Variables root

```css
:root {
  --primary  : #922B21;   /* Couleur principale — boutons, titres, barres */
  --secondary: #c0392b;   /* Couleur secondaire — dégradés */
  --success  : #48bb78;   /* Vert — bonne réponse */
  --error    : #f56565;   /* Rouge — mauvaise réponse */
  --warning  : #ed8936;   /* Orange — encadrés d'avertissement */
  --bg       : #f7fafc;
  --text     : #2d3748;
  --shadow   : 0 10px 30px rgba(0,0,0,0.15);
}
```

> 💡 Pour recolorer toute l'interface : modifier uniquement `--primary` et `--secondary`.

### CSS — Classes pédagogiques (leçons)

| Classe | Rendu | Usage |
|---|---|---|
| `.lesson-rule` | Fond blanc + trait gauche coloré | Bloc de règle avec `<h4>` |
| `.lesson-table` | Tableau avec entête coloré | Tableau comparatif |
| `.lesson-warning` | Fond jaune + trait orange | Erreur fréquente ⚠️ |
| `.lesson-example` | Fond bleu lavande `#EEF2FF` + trait `#667eea` | Exemple 💡 |

> ⚠️ `.lesson-example` est délibérément **bleu** (pas vert) pour se distinguer visuellement du feedback "bonne réponse" (vert).

### CSS — Responsive mobile

Deux `@media (max-width: 600px)` :
1. Adapte l'intro de leçon (`#lessonIntro`, `#lessonIntroContent`)
2. Fixe la largeur de la première colonne des tableaux (50px) + active `overflow-x: auto` sur `.lesson-table`

---

### Sections HTML (écrans du jeu)

| `id` | Rôle | Affiché par |
|---|---|---|
| `accueil` | Stats joueur + export/import | Défaut, `showSection('accueil')` |
| `lecons` | Liste des leçons dépliables | `showSection('lecons')` |
| `niveaux` | Grille des niveaux + sélecteur de mode | `showSection('niveaux')` |
| `jeu` | Question en cours + feedback | `startLevel(N)` |
| `resultats` | Score final + boutons | `showResults()` |

---

### LESSONS_DATA — Structure

```javascript
const LESSONS_DATA = [
  {
    num    : 1,
    title  : "Titre de la leçon",
    content: `
      <div class="lesson-rule">
        <h4>TITRE DE LA RÈGLE</h4>
        <table class="lesson-table">...</table>
      </div>
      <div class="lesson-warning">⚠️ Attention : ...</div>
      <div class="lesson-example">💡 Exemple : ...</div>
    `
  }
];
```

- `num` est indicatif (affichage) — il correspond au numéro de niveau associé
- `content` est du HTML libre : utiliser les classes `.lesson-rule`, `.lesson-table`, `.lesson-warning`, `.lesson-example`
- La leçon s'affiche automatiquement dans `#lessonIntro` au début du niveau correspondant

---

### QUESTIONS_DB — Structure

```javascript
const QUESTIONS_DB = {
  1: {
    title    : "Titre Niveau 1 🎯",
    objective: "Ce qu'on apprend dans ce niveau",
    qcm: [
      {
        q          : "Question à choix multiple ?",
        options    : ["Option A", "Option B", "Option C", "Option D"],
        correct    : "Option A",
        explanation: "Explication affichée dans le feedback."
      }
      // min 20 questions recommandées
    ],
    libre: [
      {
        q          : "Question à réponse libre ?",
        answer     : "réponse exacte",
        alternatives: ["variante acceptée", "autre variante"],  // optionnel
        explanation: "Explication."
      }
      // min 20 questions recommandées
    ]
  },
  2: { ... }
};
```

**Règles importantes :**
- `qcm[]` : le moteur tire **5 questions** en mode Mixte, **10** en mode QCM pur
- `libre[]` : **5 questions** en mode Mixte, **10** en mode Écrit pur
- `alternatives` est optionnel — les variantes sont acceptées en plus de `answer`
- La comparaison est **insensible à la casse** et **ignore la ponctuation** (`.`, `,`, `'`, `!`)

**Changer le nombre de niveaux :**
Modifier **3 endroits** dans `quiz.html` :
```javascript
// 1 — renderLevels()
for (let i = 1; i <= 5; i++)   // ← changer 5

// 2 — startNextLevel()
if (nlvl <= 5) startLevel(nlvl)  // ← changer 5

// 3 — DOMContentLoaded
if (lvl >= 1 && lvl <= 5)  // ← changer 5

// 4 — showResults() — optionnel selon la logique de déblocage
if (gameState.currentLevel < 5) p.currentLevel = ...  // ← changer 5
```

---

### gameState — Variables globales

```javascript
let gameState = {
  currentPlayer       : null,    // Prénom du joueur actif
  currentLevel        : 1,       // Niveau en cours
  currentMode         : "mixte", // "mixte" | "qcm" | "libre"
  questions           : [],      // Questions tirées pour la session
  currentQuestionIndex: 0,       // Index de la question affichée
  score               : 0,       // Score de la session en cours
  answers             : [],      // Historique des réponses données
  selectedOption      : null     // Option QCM sélectionnée non validée
};
```

---

### Flux principal du jeu

```
startLevel(N)
  → showLessonIntro(N)    ← affiche la leçon avant les questions
  → startQuestions()      ← joueur clique "C'est parti !"
  → renderQuestion()
       → QCM  : affiche boutons .option-btn
       → Libre: affiche <input type="text">
  → validateAnswer()
       → calcule isCorrect (normalisation casse + ponctuation)
       → affiche feedback .correct-fb / .wrong-fb
       → trackError() si mauvaise réponse
       → saveActiveSession() → localStorage
       → affiche bouton "Suivant" ou "Voir les résultats"
  → nextQuestion()   ← retour à renderQuestion() ou showResults()
  → showResults()
       → Math.min(correct, total) sécurise le %
       → sauvegarde sessionHistory
       → débloque niveau suivant si pct >= 80
       → savePlayers() systématiquement
```

---

### Fonctions joueur

| Fonction | Rôle |
|---|---|
| `getPlayers()` | Lit tous les joueurs depuis localStorage (try/catch) |
| `savePlayers(p)` | Écrit tous les joueurs dans localStorage |
| `getPlayerData(name)` | Retourne un joueur, le crée s'il n'existe pas |
| `switchPlayer(name)` | Active un joueur, met à jour gameState + affichage |
| `showNewPlayerModal()` | Ouvre la modale de création |
| `confirmNewPlayer()` | Valide la création du joueur |
| `deleteCurrentPlayer()` | Supprime le joueur actif après confirmation |
| `updatePlayerDisplay()` | Recharge le `<select>` + les stats du header |

---

### Navigation URL (passage du joueur entre pages)

```
quiz.html → dashboard.html : goToDashboard()
  → window.location.href = "dashboard.html?player=Prénom"

dashboard.html → quiz.html : goToQuiz()
  → window.location.href = "quiz.html?player=Prénom"

dashboard.html → niveau direct : launchLevel(N)
  → window.location.href = "quiz.html?level=N&player=Prénom"
```

Dans `DOMContentLoaded` de **quiz.html** : `params.get('player')` est lu **en priorité** avant le premier joueur de la liste.
Dans `loadPlayerList()` de **dashboard.html** : même logique, le joueur URL prend le dessus.

---

## 📄 dashboard.html — Structure détaillée

### Variables de config (en haut du `<script>`)

```javascript
const STORAGE_KEY = "histoireMaster_v1";      // même que quiz.html
const PLAYERS_KEY = "histoireMaster_players"; // même que quiz.html

const LEVEL_NAMES = {
  1: "Préhistoire & Antiquité 🦴",
  2: "Moyen Âge ⚔️",
  // ...
};
```

### BADGES_DEF — Ajouter un badge

```javascript
{
  id   : 31,
  icon : "🦊",
  name : "Nom du badge",
  desc : "Condition affichée au joueur",
  check: p => (p.score || 0) >= 3000
}
```

Le paramètre `p` est l'objet joueur depuis localStorage.
Propriétés utilisables : `p.score`, `p.completed`, `p.streak`, `p.totalQuestions`, `p.totalCorrect`, `p.cameleonHelped`, `p.errorHistory`, `p.sessionHistory`.

### CAMELEON_EXERCISES — Ajouter un exercice

```javascript
{
  error  : "Phrase incorrecte de Lucas.",
  correct: "La phrase correcte.",        // ← doit correspondre exactement à une option
  rule   : "Explication de la règle.",
  options: [
    "La phrase correcte.",    // ← même texte que correct
    "Option fausse 1",
    "Option fausse 2",
    "Option fausse 3"
  ]
}
```

> ⚠️ La valeur de `correct` doit être **exactement identique** (caractère pour caractère) à l'une des `options`.
> La correction des apostrophes est gérée en JS via `currentCamOptions[]` — pas d'attribut `onclick` avec du texte brut.

### Onglets dashboard

```javascript
function showTab(id, btn) {
  // Recharge les données fraîches depuis localStorage avant d'afficher
  // Appelle renderCameleon() si id === "cameleon"
  // Appelle renderFiches()   si id === "fiches"
}
```

---

## 📄 lexique.html — Structure détaillée

### Structure d'un mot

```javascript
const LEXIQUE = [
  {
    main : "Terme principal",       // mot, date, notion, événement...
    trans: "Traduction / def courte",
    def  : "Définition complète.",
    ex   : "Exemple dans une phrase.",
    level: 3,                        // numéro de niveau (1 à N)
    cat  : "catégorie"               // verbe, date, événement, relief...
  }
];
```

### Paramètres URL acceptés

- `lexique.html?level=3` → filtre automatiquement sur le niveau 3 au chargement
- `lexique.html#montblanc` → pré-remplit la barre de recherche avec "montblanc"
- Ces deux paramètres sont **combinables**

---

## 🔄 Créer une nouvelle matière (procédure complète)

1. Créer un nouveau dépôt GitHub (ex : `Sciences-Master`)
2. Copier les 4 fichiers du dossier `template/`
3. **Dans `quiz-template.html`** — modifier `SUBJECT_CONFIG` :
   ```javascript
   name         : "Sciences Master",
   emoji        : "🔬",
   storageKey   : "sciencesMaster_v1",    // clé UNIQUE
   playersKey   : "sciencesMaster_players" // clé UNIQUE
   ```
4. Remplacer `LESSONS_DATA` par les leçons de la nouvelle matière
5. Remplacer `QUESTIONS_DB` par les questions (même structure JSON)
6. Mettre à jour les compteurs de niveaux dans le JS (3 endroits — voir section QUESTIONS_DB)
7. Adapter les couleurs CSS (`--primary` et `--secondary`)
8. **Dans `dashboard-template.html`** — mettre les mêmes `STORAGE_KEY` et `PLAYERS_KEY`
9. Remplir `LEVEL_NAMES`, `BADGES_DEF` (noms à adapter), `CAMELEON_EXERCISES`
10. **Dans `lexique-template.html`** — remplacer `LEXIQUE[]` par le vocabulaire de la matière
11. **Dans `index-template.html`** — adapter titre, slogan, pills de stats, couleurs

---

## 🐛 Bugs connus et correctifs appliqués

### Bug tirets longs (Safari / iPhone)
**Symptôme :** Boutons gris, interface sans couleur.
**Cause :** L'iPhone substitue `--` par `–` (tiret long typographique) lors du copier-coller depuis Word ou certaines apps. Safari refuse `var(–primary)`.
**Correctif :** Toujours vérifier avec `grep "var(–" fichier.html` avant mise en ligne. Remplacer `–` par `--`.

### Bug DOMContentLoaded dupliqué
**Symptôme :** Le jeu se charge mais le joueur n'est pas retrouvé / sauvegarde perdue.
**Cause :** Code dupliqué après la fermeture `});` du `DOMContentLoaded`.
**Correctif :** Un seul `document.addEventListener("DOMContentLoaded", ...)` dans le fichier. Vérifier avec `grep -c "DOMContentLoaded" fichier.html` → doit retourner `1`.

### Bug backticks Markdown dans le HTML
**Symptôme :** Texte affiché brut avec des \`\`\` dans l'intro de leçon.
**Cause :** Artefact de copier-coller depuis un éditeur Markdown.
**Correctif :** Supprimer tous les blocs \`\`\`...\`\`\` parasites du HTML. Vérifier avec `grep -c '\`\`\`' fichier.html` → doit retourner `0`.

### Bug showResults — pourcentage > 100%
**Symptôme :** Score affiché à 110% ou plus.
**Cause :** `answers[]` pouvait contenir plus d'entrées que de questions.
**Correctif appliqué :**
```javascript
const correct = Math.min(gameState.answers.filter(a => a.isCorrect).length, total);
const pct     = Math.round((correct / total) * 100);
```

### Bug apostrophes Caméléon (bouton "Suivant" inactif)
**Symptôme :** Cliquer sur une option du Caméléon ne fait rien.
**Cause :** Les apostrophes dans les textes des options cassaient l'attribut `onclick="checkCameleon('texte avec apostrophe')"`.
**Correctif appliqué :** Stockage des options dans `currentCamOptions[]` et passage par index entier :
```javascript
// Rendu
options.map((opt, i) => `<button onclick="checkCameleon(${i})">${opt}</button>`)
// Validation
function checkCameleon(idx) { const chosen = currentCamOptions[idx]; ... }
```

---

## ✅ Bonnes pratiques — Récapitulatif

| Pratique | Détail |
|---|---|
| Variables CSS `--primary` | Recolorer toute l'interface en 2 lignes |
| `flex-wrap: wrap` systématique | Responsive sans media queries multiples |
| `grid auto-fill minmax` | Grille fluide sans breakpoints |
| localStorage structuré | Sauvegarde exportable/importable JSON |
| Comparaison normalisée (casse + ponctuation) | Réponses libres tolérantes |
| Zéro dépendance externe | Fonctionne offline, sans CDN |
| Joueur passé dans l'URL (`?player=`) | Passage transparent quiz ↔ dashboard |
| `Math.min(correct, total)` | Sécurise le calcul du pourcentage |
| `currentCamOptions[]` | Évite les bugs d'apostrophes dans le Caméléon |
| Un seul `DOMContentLoaded` | Évite les doubles initialisations |

---

*Suite Master Quiz — Architecture locale, 4 fichiers, 0 dépendance.*
