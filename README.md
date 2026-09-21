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
       function role() {
         return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
       }
       function childIdAutorise() {
         return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.childId;
       }
       function estParent() {
         return request.auth != null && role() == 'parent';
       }
       function estProprioEnfant(childId) {
         return request.auth != null && role() == 'child' && childIdAutorise() == childId;
       }

       match /users/{uid} {
         allow read: if request.auth != null && request.auth.uid == uid;
         allow write: if false; // géré uniquement depuis la console Firebase, jamais depuis l'app
       }

       match /children/{childId} {
         allow read: if estParent() || estProprioEnfant(childId);
         allow write: if estParent();
       }
     }
   }
   ```

   Ces règles imposent d'être connecté (voir étape 4). Un compte "parent" a un accès
   complet à tous les enfants ; un compte "child" ne peut que lire le profil qui lui
   est lié (aucune écriture possible). Les documents de la collection `users` ne sont
   jamais modifiables depuis l'app, uniquement depuis la console Firebase — cela évite
   qu'un compte puisse s'auto-attribuer le rôle "parent".

4. Activer l'authentification : menu **Authentication** > **Get started** > onglet
   **Sign-in method** > activer le fournisseur **Email/Password**.

5. Retourner dans **Paramètres du projet** (icône ⚙️) > **Vos applications** > **</> Web**,
   déclarer une application (nom libre, pas besoin d'hébergement Firebase), puis copier
   l'objet `firebaseConfig` généré (`apiKey`, `authDomain`, `projectId`, etc.).

## 2. Configurer l'application

Ouvrir `suivi-scolaire.html`, repérer la constante `firebaseConfig` (dans la balise
`<script>`, juste après `window.onload`) et remplacer les valeurs `YOUR_...` par
celles copiées à l'étape précédente. Enregistrer et commiter le fichier.

Tant que cette configuration n'est pas renseignée, l'app affiche un message
d'avertissement au lieu de charger les données.

## 3. Créer les comptes (parents et enfants)

Chaque personne (parent ou enfant) doit avoir son propre compte, avec des droits
différents : un **parent** peut tout gérer (ajouter/modifier/supprimer des profils et
des notes) ; un **enfant** ne peut que consulter son propre profil, en lecture seule.

Il n'y a pas d'auto-inscription dans l'app : chaque compte est créé manuellement par
vous, l'administrateur de l'app, directement dans la console Firebase.

**A. Créer le compte de connexion (Authentication)**

1. Console Firebase > **Authentication** > onglet **Users** > **Add user**.
2. Renseigner un email et un mot de passe pour la personne (parent ou enfant),
   puis valider.
3. Noter l'**UID** généré pour cet utilisateur (colonne "User UID") — il sera
   nécessaire à l'étape suivante.

**B. Déclarer son rôle (Firestore)**

1. Console Firebase > **Firestore Database** > onglet **Données**.
2. Créer (ou ouvrir) la collection `users`.
3. Ajouter un document dont l'**ID du document est exactement l'UID** noté à l'étape A :
   - Pour un **parent** : un seul champ `role` (string) = `parent`.
   - Pour un **enfant** : un champ `role` (string) = `child`, et un champ `childId`
     (string) = l'ID du document de cet enfant dans la collection `children`
     (visible dans l'onglet Données > `children` > cliquer sur le profil concerné,
     l'ID est affiché en haut de la fiche du document).

Répéter A et B pour chaque parent et chaque enfant devant se connecter. La personne
se connecte ensuite sur l'app avec l'email et le mot de passe créés à l'étape A.

## 4. Déployer (GitHub Pages)

1. Pousser les fichiers (`index.html`, `suivi-scolaire.html`) sur la branche par défaut du dépôt.
2. Dans les paramètres du dépôt GitHub : **Settings > Pages**.
3. En **Source**, choisir **Deploy from a branch**, sélectionner la branche
   principale et le dossier `/ (root)`, puis **Save**.
4. Après quelques minutes, l'app est accessible à l'URL fournie par GitHub
   (ex: `https://<utilisateur>.github.io/<repo>/`).

## Fonctionnalités

- Connexion par compte (email + mot de passe), avec deux rôles :
  - **Parent** : accès complet à tous les enfants (ajout, édition, suppression de profils et de notes).
  - **Enfant** : accès en lecture seule à son propre profil uniquement (pas de suppression, pas de modification, pas de visibilité sur les autres enfants).
- Gestion de plusieurs profils d'enfants (ajout, édition, suppression) — réservé au rôle parent.
- Saisie des notes avec matière, coefficient, et calcul automatique de la moyenne générale.
- Objectif de moyenne réglable, avec projection du nombre de points nécessaires.
- Simulateur d'impact d'un futur devoir sur la moyenne.
- Liens de révision (Lumni, Kartable) préremplis selon le niveau et la matière.
- Données synchronisées en direct entre tous les appareils via Firestore.
