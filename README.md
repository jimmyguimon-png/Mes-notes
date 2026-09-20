# Mes-notes — Suivi Scolaire

Application web (fichier HTML unique) pour permettre à des parents de suivre les notes,
la moyenne et les objectifs scolaires de leurs enfants, sur tous leurs appareils.

## 1. Créer le projet Firebase (base de données)

L'app a besoin d'une base Firestore pour sauvegarder les profils et les notes,
et les synchroniser en direct entre les appareils (téléphone du parent, tablette de l'enfant, etc.).

1. Aller sur [console.firebase.google.com](https://console.firebase.google.com) et créer un nouveau projet (gratuit).
2. Dans le menu de gauche, ouvrir **Firestore Database** > **Créer une base de données**.
   - Choisir une région proche (ex: `eur3 (europe-west)`).
   - Démarrer en **mode production**.
3. Aller dans **Règles** de Firestore et coller les règles suivantes, puis **Publier** :

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /children/{childId} {
         allow read, write: if true;
       }
     }
   }
   ```

   ⚠️ Ces règles ouvrent la collection `children` à quiconque connaît l'URL du projet
   (pas de compte à créer, adapté à un usage familial). Ne mettez rien de sensible
   (aucune donnée médicale, financière, etc.) dans les notes ou noms saisis.

4. Retourner dans **Paramètres du projet** (icône ⚙️) > **Vos applications** > **</> Web**,
   déclarer une application (nom libre, pas besoin d'hébergement Firebase), puis copier
   l'objet `firebaseConfig` généré (`apiKey`, `authDomain`, `projectId`, etc.).

## 2. Configurer l'application

Ouvrir `suivi-scolaire.html`, repérer la constante `firebaseConfig` (dans la balise
`<script>`, juste après `window.onload`) et remplacer les valeurs `YOUR_...` par
celles copiées à l'étape précédente. Enregistrer et commiter le fichier.

Tant que cette configuration n'est pas renseignée, l'app affiche un message
d'avertissement au lieu de charger les données.

## 3. Déployer (GitHub Pages)

1. Pousser les fichiers (`index.html`, `suivi-scolaire.html`) sur la branche par défaut du dépôt.
2. Dans les paramètres du dépôt GitHub : **Settings > Pages**.
3. En **Source**, choisir **Deploy from a branch**, sélectionner la branche
   principale et le dossier `/ (root)`, puis **Save**.
4. Après quelques minutes, l'app est accessible à l'URL fournie par GitHub
   (ex: `https://<utilisateur>.github.io/<repo>/`).

## Fonctionnalités

- Gestion de plusieurs profils d'enfants (ajout, édition, suppression).
- Saisie des notes avec matière, coefficient, et calcul automatique de la moyenne générale.
- Objectif de moyenne réglable, avec projection du nombre de points nécessaires.
- Simulateur d'impact d'un futur devoir sur la moyenne.
- Liens de révision (Lumni, Kartable) préremplis selon le niveau et la matière.
- Données synchronisées en direct entre tous les appareils via Firestore.
