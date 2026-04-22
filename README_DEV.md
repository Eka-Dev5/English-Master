# 📘 README DÉVELOPPEUR — English Master

> Architecture, bonnes pratiques et guide de modification du projet.

---

## 🗂️ Architecture générale

Le projet repose sur **3 fichiers HTML autonomes** qui communiquent uniquement via le `localStorage` du navigateur. Aucun serveur, aucune base de données, aucune dépendance externe.

```
quiz.html          ← Point d'entrée principal (jeu)
dashboard.html     ← Espace personnel joueur
lexique.html       ← Vocabulaire interactif
```

Les 3 fichiers doivent **impérativement être dans le même dossier** pour que les liens internes fonctionnent.

---

## 🔑 Clés localStorage partagées

| Clé | Contenu | Fichiers concernés |
|---|---|---|
| `englishMaster_players` | Objet JSON de tous les joueurs | quiz.html + dashboard.html |
| `englishMaster_v4` | (héritage — non utilisé en v5) | — |

Structure d'un joueur dans `englishMaster_players` :
```json
{
  "Prénom": {
    "name": "Prénom",
    "currentLevel": 3,
    "score": 120,
    "completed": [1, 2],
    "totalQuestions": 45,
    "totalCorrect": 38,
    "streak": 5,
    "lastPlayed": "2025-04-22T10:30:00.000Z",
    "errorHistory": [...],
    "cameleonHelped": 2
  }
}
```

---

## 📄 quiz.html — Structure détaillée

### HEAD

```html
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```
Le viewport mobile-first est essentiel pour le rendu sur tablette/téléphone (usage CM2).

### CSS — Variables root

```css
:root {
  --primary: #667eea;
  --secondary: #764ba2;
  --success: #48bb78;
  --error: #f56565;
  --warning: #ed8936;
}
```
**Bonne pratique appliquée :** toutes les couleurs passent par des variables CSS. Pour changer le thème complet du jeu, modifier ces 5 valeurs suffit. Aucune couleur codée en dur dans les règles de composants.

### CSS — Layout

- `display: flex` avec `flex-wrap: wrap` sur tous les conteneurs de boutons → responsive automatique sans media query redondante.
- `grid-template-columns: repeat(auto-fill, minmax(230px, 1fr))` sur la grille des niveaux → s'adapte de 1 à N colonnes selon la largeur disponible, sans breakpoints manuels.
- `gap` systématique au lieu de `margin` → évite les collisions de marges et simplifie le spacing.

### CSS — Animations

```css
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(-8px); }
  to   { opacity: 1; transform: translateY(0); }
}
```
Utilisée sur l'affichage du feedback. Légère, CSS-only, sans librairie.

### CSS — Media query

```css
@media (max-width: 600px) {
  .player-selector { flex-direction: column; align-items: stretch; }
  .player-selector select { width: 100%; }
}
```
Seul point de rupture nécessaire : empile le sélecteur de joueur verticalement sur mobile.

---

### SUBJECT_CONFIG — Bloc de configuration sujet

```javascript
const SUBJECT_CONFIG = {
  name: "English Master",
  emoji: "🎓",
  storageKey: "englishMaster_v4",
  playersKey: "englishMaster_players",
  dashboardFile: "dashboard.html",
  lexiqueFile: "lexique.html"
};
```
**Point d'entrée principal pour adapter le jeu à une autre matière.** Modifier `name` et `playersKey` (clé unique pour ne pas mélanger les sauvegardes d'une matière à l'autre).

---

### LESSONS_DATA — Structure d'une leçon

```javascript
{
  num: 1,
  title: "TO BE / TO HAVE (Être / Avoir)",
  content: `<div class="lesson-rule">
    <h4>Titre de la règle</h4>
    <table class="lesson-table">...</table>
  </div>
  <div class="lesson-warning">⚠️ Attention : ...</div>
  <div class="lesson-example">✅ Exemples ...</div>`
}
```

Classes disponibles dans `content` :

| Classe | Usage |
|---|---|
| `.lesson-rule` | Bloc de règle avec titre |
| `.lesson-table` | Tableau comparatif |
| `.lesson-warning` | Encadré avertissement (fond jaune) |
| `.lesson-example` | Encadré exemple (fond vert) |

**Pour ajouter une leçon :** ajouter un objet à la fin du tableau `LESSONS_DATA`. Le numéro `num` est indicatif (affichage), il n'est pas lié à un niveau de quiz.

---

### QUESTIONS_DB — Structure d'un niveau

```javascript
5: {
  title: "Retour en arrière ⏪",
  objective: "Prétérit Was / Were",
  qcm: [
    {
      q: "I ___ happy yesterday.",
      options: ["was", "were", "am", "is"],
      correct: "was",
      explanation: "I + WAS (prétérit de BE)."
    },
    // ... 19 autres questions
  ],
  libre: [
    {
      q: "I (be) ___ happy yesterday.",
      answer: "was",
      alternatives: ["'s"],   // optionnel : réponses acceptées en plus
      explanation: "I + WAS."
    },
    // ... 19 autres questions
  ]
}
```

**Règles importantes :**
- La clé numérique (`5:`) correspond au numéro de niveau (1 à 10).
- `qcm` : 20 questions minimum recommandées. Le moteur en tire 5 aléatoirement en mode Mixte, 10 en mode QCM.
- `libre` : même logique, 10 en mode Écrit, 5 en Mixte.
- `alternatives` est optionnel. Si présent, les réponses listées sont acceptées en plus de `answer`.
- La comparaison est insensible à la casse et ignore la ponctuation (`.`, `,`, `'`, `!`).

**Pour ajouter des questions à un niveau existant :**
```javascript
// Trouver le niveau dans QUESTIONS_DB, ex: niveau 3
3: {
  title: "...",
  objective: "...",
  qcm: [
    // ... questions existantes ...
    // Ajouter ici :
    {q: "Nouvelle question ___.", options: ["A","B","C","D"], correct: "A", explanation: "Explication."}
  ],
  libre: [
    // ... questions existantes ...
    {q: "Nouvelle question (verbe) ___.", answer: "réponse", explanation: "Explication."}
  ]
}
```

**Pour créer un niveau 11 :**
```javascript
// Dans QUESTIONS_DB, après le niveau 10, ajouter :
11: {
  title: "Nouveau thème 🆕",
  objective: "Description courte",
  qcm: [ /* 20 questions */ ],
  libre: [ /* 20 questions */ ]
}
```
Puis dans `renderLevels()`, changer `i <= 10` en `i <= 11`.

---

### Logique de jeu — Flux principal

```
startLevel(N)
  → shuffle questions selon le mode
  → showSection('jeu')
  → renderQuestion()
       → si qcm  : affiche boutons options
       → si libre : affiche input texte
  → validateAnswer()
       → calcule isCorrect
       → affiche feedback
       → met à jour localStorage
       → affiche bouton "Suivant"
  → nextQuestion() ou showResults()
```

---

## 📄 dashboard.html — Structure détaillée

### Onglets

La navigation par onglets est gérée par `showTab(id, btn)` :
```javascript
function showTab(id, btn) {
  document.querySelectorAll(".tab-content").forEach(t => t.classList.remove("active"));
  document.querySelectorAll(".tab-btn").forEach(b => b.classList.remove("active"));
  document.getElementById(id).classList.add("active");
  if (btn) btn.classList.add("active");
}
```
Simple, sans librairie. L'onglet actif est stylisé via `.tab-btn.active`.

### Badges — Ajouter un badge

```javascript
// Dans BADGES_DEF, ajouter un objet :
{
  id: 31,
  icon: "🦊",
  name: "Nom du badge",
  desc: "Condition affichée au joueur",
  check: p => (p.score || 0) >= 3000   // logique de déverrouillage
}
```
Le paramètre `p` est l'objet joueur depuis localStorage. Toutes les propriétés du joueur sont accessibles (`p.score`, `p.completed`, `p.streak`, `p.totalQuestions`, `p.cameleonHelped`, etc.).

### Mode Caméléon — Ajouter des exercices

```javascript
// Dans CAMELEON_EXERCISES, ajouter :
{
  error: "Phrase avec l'erreur de Lucas.",
  correct: "Phrase correcte.",
  rule: "Explication de la règle à retenir.",
  options: [
    "Option 1 (fausse)",
    "Option 2 (correcte)",
    "Option 3 (fausse)",
    "Option 4 (fausse)"
  ]
}
```
L'option correcte doit correspondre exactement à la valeur de `correct`.

---

## 📄 lexique.html — Structure détaillée

### Ajouter des mots

```javascript
// Dans le tableau LEXIQUE, ajouter :
{
  en: "mot en anglais",
  fr: "traduction française",
  def: "Définition courte et claire.",
  ex: "Exemple d'utilisation dans une phrase.",
  level: 3,        // niveau auquel ce mot appartient (1-10)
  cat: "verbe"     // catégorie : verbe, nom, adjectif, adverbe, modal, expression, conjonction, préposition
}
```

---

## 🔄 Adapter à une autre matière

**Exemple : Maths Master**

1. Dupliquer les 3 fichiers, renommer `quiz.html` → `maths-quiz.html`, etc.
2. Dans `quiz.html`, modifier `SUBJECT_CONFIG` :
```javascript
const SUBJECT_CONFIG = {
  name: "Maths Master",
  emoji: "🔢",
  playersKey: "mathsMaster_players",   // clé différente = sauvegardes séparées
  dashboardFile: "maths-dashboard.html",
  lexiqueFile: "maths-lexique.html"
};
```
3. Remplacer `LESSONS_DATA` par les leçons de maths.
4. Remplacer `QUESTIONS_DB` par les questions de maths (même structure JSON).
5. Dans `lexique.html`, remplacer le tableau `LEXIQUE` par le vocabulaire mathématique.
6. Dans `dashboard.html`, remplacer `CAMELEON_EXERCISES` par des erreurs typiques en maths.

La clé `playersKey` différente garantit que les progressions ne se mélangent pas entre les matières.

---

## ✅ Bonnes pratiques appliquées — Récapitulatif

| Pratique | Détail |
|---|---|
| Variables CSS (`--primary`, etc.) | Thème modifiable en 5 lignes |
| `flex-wrap: wrap` systématique | Responsive sans media queries multiples |
| `grid auto-fill minmax` | Grille fluide sans breakpoints |
| Fonctions courtes et nommées | Lisibilité et maintenance facilitées |
| `localStorage` structuré | Sauvegarde robuste, exportable |
| Comparaison de réponse normalisée | Insensible à casse et ponctuation |
| Zéro dépendance externe | Fonctionne offline, sans CDN |
| HTML sémantique | `<button>`, `<input>`, `<select>` natifs |
| Animations CSS-only | Légères, sans JS ni librairie |
| Config sujet isolée | Duplication rapide pour autre matière |

---

*Projet English Master — Architecture locale, 3 fichiers, 0 dépendance.*
