# BATAILLE — V1

Jeu de quiz compétitif. « Qui est le plus fort ? »

## Architecture : pages statiques multiples (pas de SPA)

Décision prise à l'étape 2 : plutôt qu'un routeur SPA (prévu initialement),
le site utilise plusieurs pages HTML statiques (`index.html`, `auth.html`,
puis `theme-select.html`, `game.html`, etc. aux étapes suivantes). C'est
plus simple à maintenir pour un développeur solo, tout aussi compatible
avec Vercel, et évite de construire un routeur maison pour un site qui
reste petit.

## Architecture : fonctions serverless Vercel plutôt que Cloud Functions Firebase

Décision prise pour rester 100% gratuit sans carte bancaire (les Cloud
Functions Firebase exigent le plan payant Blaze) : la logique serveur
(sélection des questions, validation des réponses) vit dans `/api`
(fonctions serverless Vercel, gratuites, déjà utilisées pour l'hébergement)
et utilise le SDK Admin Firebase pour parler à Firestore. Voir
`lib/firebaseAdmin.js` pour la configuration requise.

## Importer les questions sans terminal (alternative simple)

Plutôt que `scripts/import-questions.js` (nécessite Node + firebase-admin en
local), tu peux utiliser `/api/admin-seed-questions` : une fois le projet
déployé sur Vercel avec `FIREBASE_SERVICE_ACCOUNT_KEY` **et**
`ADMIN_SEED_SECRET` définies (voir `.env.example`), ouvre simplement cette
URL dans ton navigateur :

```
https://<ton-projet>.vercel.app/api/admin-seed-questions?secret=<ton ADMIN_SEED_SECRET>
```

Ça importe les 48 questions de test en une requête, sans rien installer.
Redéployable à volonté (ça écrase les questions existantes avec les mêmes
id, sans dupliquer).

## Config Firebase réelle renseignée

`src/firebase/config.js` contient désormais la vraie config du projet Firebase (`bataille-60f91`). Il reste à :
1. Activer Firestore (mode natif) et Authentication (E-mail/Mot de passe + Anonyme) dans la console
2. Déployer `firestore.rules`
3. Générer une clé de compte de service et définir `FIREBASE_SERVICE_ACCOUNT_KEY` sur Vercel
4. Importer les questions de test (`npm run import:questions`)

## État actuel (étape 8 — profil + classement)

Ce qui est **réellement fonctionnel** :
- Tout ce des étapes 1 à 7
- `/api/finish-game` suit maintenant aussi les **victoires/défaites** (uniquement pour les parties de défi) — mis à jour pour les deux joueurs dans la même transaction que le score/XP, y compris le profil de l'adversaire
- `profile.html` : pseudo, niveau avec barre XP, parties jouées, meilleur score, victoires, défaites, série — toutes des stats **réellement stockées**, aucune donnée inventée
- `leaderboard.html` : top 10 par XP + position exacte du joueur (via une requête d'agrégation Firestore, pas un comptage manuel)
- Liens "Classement" / "Profil" ajoutés à la navigation (accueil, choix du thème, défi)
- **Choix d'architecture** : pas de collection `leaderboard` séparée comme évoqué dans le cadrage initial — le classement interroge directement `users` trié par XP (lecture publique déjà autorisée par les règles), ce qui évite une deuxième source de vérité à synchroniser. Section 23 du prompt maître autorise explicitement à adapter la structure.
- **Correctif de sécurité appliqué en cours de route** : le pseudo (choisi librement par l'utilisateur) était injecté via `innerHTML` dans le header et le classement — risque de XSS stocké. Corrigé partout en passant par `textContent`/DOM plutôt que par de l'HTML interpolé, avant toute mise en ligne.

⚠️ Toujours pas testé de bout en bout avec un vrai déploiement.

Ce qui **n'est pas encore implémenté** :
- BATAILLE MIX, QR code pour rejoindre un défi, classements hebdo/mensuel/par thème (prévus "ultérieurement" par le prompt maître, pas dans cette V1)

## État actuel (étape 9 — BATAILLE MIX)

Ce qui est **réellement fonctionnel** :
- Tout ce des étapes 1 à 8
- `lib/selectQuestions.js` : `selectMixQuestions()` applique la répartition fixe du prompt maître (2 Côte d'Ivoire, 2 Football, 2 Musique, 2 Culture générale, 1 Drapeaux & pays, 1 Sciences & techno = 10 questions) — **la somme est vérifiée (10)**, et l'ordre final est re-mélangé pour ne pas grouper les questions par thème
- `/api/start-game` accepte désormais `category: "mix"` en plus des 6 thèmes classiques, en réutilisant exactement la même mécanique de partie (anti-triche, score, XP) que le mode solo normal — aucune duplication de logique
- La carte BATAILLE MIX sur `theme-select.html` est activée et lance une vraie partie

⚠️ Le mode MIX reste réservé aux parties solo pour cette V1 (pas de défi en MIX) — cohérent avec le prompt maître qui ne mentionne le mode mix que pour le jeu normal, pas pour les défis.

⚠️ Toujours pas testé de bout en bout avec un vrai déploiement — vérifié par relecture, validation de syntaxe, et test isolé de la répartition (10 questions confirmées).

Ce qui **n'est pas encore implémenté** :
- QR code pour rejoindre un défi (uniquement "si possible sans complexifier la V1" selon le prompt maître — à évaluer)

## État actuel (étape 10 — responsive + polish)

Ce qui a été fait :
- **Hover desktop** : boutons et cartes cliquables réagissent au survol (souris réelle uniquement — `@media (hover: hover) and (pointer: fine)`, jamais sur tactile pour éviter l'état "collé" après un tap)
- **Animation d'entrée** légère sur chaque page (`page-fade-in` sur `.container`) — subtile, respecte `prefers-reduced-motion` (déjà neutralisée globalement pour qui le demande)
- **Feedback de réponse animé** : les boutons corrects/incorrects ont maintenant un petit "pulse" au lieu d'un simple changement de couleur statique
- **Grilles responsives** : catégories (accueil), thèmes (choix du thème), mini-thèmes (défi), stats (profil) passent de 2 colonnes (mobile) à 3-4 colonnes à partir de 640px — plus rien ne reste écrasé sur tablette/desktop alors que le design reste mobile-first
- **Zones de sécurité (encoche/barre iPhone)** : `env(safe-area-inset-*)` ajouté sur les conteneurs et le bouton Continuer de l'écran de jeu
- `-webkit-tap-highlight-color: transparent` — plus de flash gris disgracieux au tap sur mobile
- Clavier mobile en majuscules automatiques sur le champ de code de bataille
- Accolades CSS vérifiées équilibrées sur tous les fichiers modifiés

⚠️ Honnêteté sur les limites de cette passe : je n'ai **aucun navigateur ni appareil réel** pour tester visuellement ici — tout ça est une relecture attentive du CSS (grilles, breakpoints, tailles de cible tactile ≥48px, media queries), pas un test sur un vrai petit smartphone/tablette/grand écran comme le demande la section 26 du prompt maître. Un aperçu HTML autonome de la page d'accueil (`bataille-apercu.html`) est fourni pour un contrôle visuel rapide, mais teste idéalement toi-même sur un vrai téléphone avant de considérer cette étape validée.

Ce qui **n'est pas encore fait** :
- Vérification des 600 questions finales (toujours 48 de test)
- Tests de sécurité Firestore en conditions réelles
- Déploiement réel

## Prochaine étape

À toi de jouer : déployer et tester en conditions réelles (voir les instructions plus haut), ou continuer vers la rédaction des questions manquantes.

Voir `bataille-v1-cadrage.md` pour le cadrage complet (architecture, Firestore, flux, risques, plan par étapes).

## Lancer en local

Ouvrir `index.html` via un petit serveur statique (ex. `npx serve .`) — les imports ES modules ne fonctionnent pas en ouvrant le fichier directement (`file://`).
