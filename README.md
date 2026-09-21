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
         allow read: if request.auth != null && (request.auth.uid == uid || estParent());
         allow write: if estParent(); // seul un parent déjà existant peut créer/gérer des comptes
       }

       match /children/{childId} {
         allow read: if estParent() || estProprioEnfant(childId);
         allow write: if estParent();
       }
     }
   }
   ```

   Ces règles imposent d'être connecté (voir étape 4). Un compte "parent" a un accès
   complet à tous les enfants ainsi qu'à la gestion des comptes ; un compte "child" ne
   peut que lire le profil qui lui est lié (aucune écriture possible). Un compte ne peut
   jamais s'auto-attribuer le rôle "parent" : il faut déjà être parent pour écrire dans
   la collection `users`, y compris pour créer son propre compte.

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
des notes, et créer d'autres comptes) ; un **enfant** ne peut que consulter son propre
profil, en lecture seule.

**Une seule étape manuelle est nécessaire : le tout premier compte parent.** Tous les
comptes suivants (autres parents, enfants) se créent ensuite directement depuis l'app,
dans l'onglet **Comptes** (visible uniquement une fois connecté avec un compte parent).

**A. Créer le tout premier compte parent (une seule fois, dans la console Firebase)**

1. Console Firebase > **Authentication** > onglet **Users** > **Add user**.
2. Renseigner votre email et un mot de passe, puis valider.
3. Noter l'**UID** généré (colonne "User UID").
4. Console Firebase > **Firestore Database** > onglet **Données** > créer la collection
   `users` > ajouter un document dont l'**ID est exactement cet UID**, avec un seul
   champ `role` (string) = `parent`.
5. Se connecter sur l'app avec cet email et ce mot de passe.

**B. Créer les autres comptes (depuis l'app, une fois connecté en tant que parent)**

1. Ouvrir l'onglet **Comptes**.
2. Cliquer sur **+ Ajouter un compte**.
3. Renseigner un email et un mot de passe pour la personne, choisir son rôle
   (**Parent** ou **Enfant**), et pour un enfant, sélectionner le profil élève à lier.
4. Cliquer sur **Créer le compte**, puis communiquer l'email et le mot de passe à la
   personne concernée pour qu'elle se connecte.

Il n'y a pas de bouton de suppression de compte dans l'app (Firebase ne permet pas à un
compte de supprimer un autre compte sans configuration serveur supplémentaire) : pour
révoquer un accès, désactivez ou supprimez l'utilisateur correspondant dans Console
Firebase > **Authentication** > **Users**.

## 4. Notification par email à chaque note (optionnel)

L'app peut envoyer un email automatiquement à chaque fois qu'une note est ajoutée ou
modifiée pour un enfant, via [EmailJS](https://www.emailjs.com) (gratuit jusqu'à 200
emails/mois, aucun serveur à héberger).

1. Créer un compte gratuit sur [emailjs.com](https://www.emailjs.com).
2. **Email Services** > **Add New Service** > connecter une adresse email (Gmail,
   Outlook...). Noter le **Service ID** généré.
3. **Email Templates** > **Create New Template**. Utiliser ces variables dans le
   contenu du modèle : `{{to_email}}`, `{{prenom}}`, `{{matiere}}`, `{{note}}`,
   `{{coeff}}`. Exemple de corps de message :
   ```
   Nouvelle note pour {{prenom}} : {{matiere}} — {{note}}/20 (coefficient {{coeff}})
   ```
   Dans les paramètres du template, définir le champ **To Email** sur `{{to_email}}`.
   Noter le **Template ID**.
4. **Account** > **General** : noter la **Public Key**.
5. Ouvrir `suivi-scolaire.html`, repérer la constante `emailjsConfig` (juste après
   `firebaseConfig`) et remplacer les valeurs `YOUR_EMAILJS_...` par celles notées
   ci-dessus.

Tant que `emailjsConfig.publicKey` vaut `YOUR_EMAILJS_PUBLIC_KEY`, aucun email n'est
envoyé (fonctionnement normal du reste de l'app inchangé). Une fois configuré, chaque
profil élève peut renseigner un "Email de notification" dans son Profil (repliable) —
laissez ce champ vide pour ne recevoir aucun email pour cet enfant.

## 5. Déployer (GitHub Pages)

1. Pousser les fichiers (`index.html`, `suivi-scolaire.html`) sur la branche par défaut du dépôt.
2. Dans les paramètres du dépôt GitHub : **Settings > Pages**.
3. En **Source**, choisir **Deploy from a branch**, sélectionner la branche
   principale et le dossier `/ (root)`, puis **Save**.
4. Après quelques minutes, l'app est accessible à l'URL fournie par GitHub
   (ex: `https://<utilisateur>.github.io/<repo>/`).

## Fonctionnalités

- Connexion par compte (email + mot de passe), avec deux rôles :
  - **Parent** : accès complet à tous les enfants (ajout, édition, suppression de profils et de notes), et peut créer d'autres comptes directement depuis l'onglet Comptes.
  - **Enfant** : accès en lecture seule à son propre profil uniquement (pas de suppression, pas de modification, pas de visibilité sur les autres enfants).
- Gestion de plusieurs profils d'enfants (ajout, édition, suppression) — réservé au rôle parent.
- Saisie des notes avec matière, coefficient, et calcul automatique de la moyenne générale.
- Objectif de moyenne réglable, avec projection du nombre de points nécessaires.
- Simulateur d'impact d'un futur devoir sur la moyenne.
- Liens de révision (Lumni, Kartable) préremplis selon le niveau et la matière.
- Données synchronisées en direct entre tous les appareils via Firestore.
- Notification par email (optionnelle, via EmailJS) à chaque note ajoutée ou modifiée.
