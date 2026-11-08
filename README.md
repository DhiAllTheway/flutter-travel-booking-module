# Travel Platform – application mobile Flutter + SQLite

Projet **5SAE3 – ESPRIT**. Application mobile de voyage organisée autour du **voyage** :
transport (commandes clients et trajets de transporteurs), location de voiture,
hébergement, forum et avis, actualités. L'interface suit le **UI Kit Travel Platform**
(turquoise `#0F7B83`, orange `#F68B3F`, police Plus Jakarta Sans).

## En bref

| Élément | Valeur |
|---|---|
| Version de l'application | `1.0.0+1` (`pubspec.yaml`) |
| Framework | Flutter, Dart `^3.5.0`, Material 3 |
| Base de données | SQLite locale, fichier `travel_platform.db`, version de schéma `1`, **14 tables** |
| Rôles | Client, Voyageur (`traveller`), Transporteur (`transporteur`), Administrateur |
| Modules | 6 dossiers dans `lib/modules/` |
| Tests Flutter | 31 tests dans `test/` (4 fichiers) |
| Tests serveur IA | 13 tests dans `server/test/` |
| Service optionnel | Serveur IA Node.js (`server/`) utilisant Claude Haiku 5.5 |

## Sommaire

1. Modules et responsables
2. Fonctionnalités détaillées
3. Architecture
4. Base de données
5. API REST et serveur IA
6. Intelligence artificielle (cahier des charges §7.2)
7. Installation (Windows 10 / 11)
8. Lancer l'application
9. Tests
10. Comptes et données de départ
11. Règles de saisie et de validation
12. Sécurité
13. Dépannage et journaux
14. Workflow d'équipe
15. État d'avancement et points ouverts
16. Structure du projet

---

## 1. Modules et responsables

| Dossier | Module | Responsable | Tables | Routes principales |
|---|---|---|---|---|
| `lib/modules/user` | Comptes, rôles, accueil, actualités, administration | Abdeslem Kessentini | `utilisateur`, `actualite` | `/`, `/onboarding`, `/login`, `/signup`, `/home`, `/admin/...` |
| `lib/modules/voyage` | Voyage | Mouhamed Dhia Dridi | `voyage` | `/voyage/liste`, `/voyage/nouveau` |
| `lib/modules/transporteur` | Transport (commandes, trajets, mises en relation, IA) | Abir Gamoudi | `transporteur`, `reservation_transport`, `commande_transport`, `mise_en_relation` | `/transport/...` |
| `lib/modules/location` | Location de voiture | Roua Dhafer | `vehicule`, `location_vehicule` | `/location`, `/location/gestion` |
| `lib/modules/hebergement` | Hébergement | Oussema Mrayah | `hebergement`, `reservation_hebergement` | `/hebergement`, `/hebergement/gestion` |
| `lib/modules/forum` | Forum et avis | Sirine Hamzaoui | `forum_sujet`, `forum_reponse`, `avis` | `/forum`, `/forum/avis` |

La répartition est reprise du cahier des charges (rapport « Travel Platform – Application Overview and Module Specification »).

---

## 2. Fonctionnalités détaillées

### 2.1 Comptes, accueil et administration (user – Abdeslem Kessentini)

- **Démarrage** : écran Splash (environ 1,4 s), puis parcours d'accueil en 3 écrans si aucune session n'est ouverte, sinon accueil.
- **Inscription** : rôles proposés Client, Voyageur et Transporteur. Le rôle Administrateur n'est jamais proposé. L'e-mail doit être unique, sans tenir compte des majuscules.
- **Connexion** : vérification du mot de passe haché (voir §12). Une session mémorise seulement l'identifiant ; le rôle est relu en base à chaque ouverture.
- **Accueil par rôle** (onglets Accueil, Actualités, Profil) :
  - Client : raccourcis, bouton orange « Nouvelle commande », tuiles du module transport.
  - Voyageur : grille « Votre voyage ».
  - Transporteur : « Mises en relation », bouton « Commandes ouvertes », tuiles transport.
  - Administrateur : « Gestion de la plateforme », bouton « Publier une actualité », tuiles d'administration. Le menu latéral ouvre la gestion de la flotte et des établissements.
- **Actualités** : liste des annonces publiées par l'administrateur, avec date en majuscules (ex. « 15 OCT. 2026 »).
- **Profil** : nom, prénom et téléphone modifiables ; e-mail et rôle en lecture seule ; déconnexion avec confirmation.
- **Administration** : publier, modifier et supprimer des actualités ; lister les utilisateurs ; supprimer un compte. Le compte administrateur ne peut jamais être supprimé, un compte client peut l'être.

### 2.2 Voyage (voyage – Mouhamed Dhia Dridi)

- Un voyage a un titre, une destination, une date de début, une date de fin, un budget et un statut : `planifie`, `actif`, `termine` ou `annule`.
- La date de fin ne peut pas être antérieure à la date de début ; un voyage d'une seule journée est accepté.
- Seul le propriétaire peut modifier ou supprimer son voyage.
- Écrans : **Mes voyages** (liste) et **Nouveau voyage** (formulaire).
- Les voyages déclarés alimentent le classement des voyageurs du module transport (voir §2.3).

### 2.3 Transport (transporteur – Abir Gamoudi)

**Transporteurs** (fiches : nom, type, contact)
- Types proposés : Bus, Train, Avion, Bateau, Covoiturage.
- Supprimer une fiche conserve ses trajets : le trajet garde son prix et perd seulement le lien vers la fiche.

**Trajets proposés** (`reservation_transport`)
- Un trajet a un départ, une arrivée (différents, contrôlé par la base), une date de départ, un prix et un statut : en attente, confirmée ou annulée.
- Recherche par départ, arrivée et date, insensible à la casse. Résultats triés par date puis par prix.

**Commandes client** (`commande_transport`)
- Une commande a une ville d'achat, une ville de livraison, une date souhaitée (non passée), un budget en dinars (positif ou nul) et des détails facultatifs.
- Statuts : ouverte, acceptée, annulée.
- Modifier, annuler ou supprimer est réservé au client propriétaire. Modifier et annuler ne sont possibles que tant que la commande est ouverte ; supprimer est interdit une fois la commande acceptée.
- Le formulaire **Nouvelle commande** peut être rempli automatiquement à partir d'une phrase (assistant IA, voir §6). Les champs restent modifiables à la main.

**Commandes ouvertes et voyageurs candidats**
- Un transporteur voit les commandes ouvertes et la liste des voyageurs, chacun avec ses voyages déclarés.
- Le classement des voyageurs pour une commande est fait par l'IA, avec repli sur un score local (voir §6).
- Une seule proposition active est possible par voyageur et par commande.

**Mises en relation** (`mise_en_relation`)
- Le transporteur propose un voyageur pour une commande ouverte. La proposition est en attente jusqu'à la réponse du voyageur.
- Accepter une proposition ferme la commande (statut acceptée, voyageur renseigné) et refuse automatiquement les autres propositions en attente pour la même commande.
- Les coordonnées (téléphone) du client et du voyageur ne sont visibles qu'après acceptation.

**Écrans** : Nouvelle commande, Mes commandes, Commandes ouvertes, Mises en relation, Réservations (liste des trajets et des transporteurs), Voyageur (recherche de trajets, propositions reçues, résumé IA d'un trajet), formulaires trajet et transporteur.

### 2.4 Location de voiture (location – Roua Dhafer)

- **Flotte** : véhicules avec marque, modèle, prix par jour et indicateur de disponibilité.
- **Réservations** : dates de début et de fin, prix total = prix par jour × nombre de jours. Les bornes sont comprises : du 10 au 12 font 3 jours.
- Une réservation peut être rattachée à un voyage ; elle est supprimée avec le voyage.
- Refus des chevauchements de dates et des véhicules indisponibles.
- Supprimer un véhicule supprime ses réservations.
- **Écrans** : Location (accueil, locations, véhicules), formulaires location et véhicule, **Gérer la flotte** (protégé par `GardeAdmin`).

### 2.5 Hébergement (hebergement – Oussema Mrayah)

- **Établissements** : nom, type, adresse, prix par nuit.
- **Séjours** : date d'arrivée, date de départ (strictement après l'arrivée), nombre de personnes (1 par défaut), prix total = prix par nuit × nombre de nuits, rattachement facultatif à un voyage.
- Supprimer un établissement supprime ses séjours.
- **Écrans** : Hébergement (établissements, séjours), formulaires établissement et séjour, **Gérer les établissements** (protégé par `GardeAdmin`).

### 2.6 Forum et avis (forum – Sirine Hamzaoui)

- **Sujets** : titre, contenu, auteur, date. Supprimer un sujet supprime ses réponses.
- **Réponses** à un sujet.
- **Avis** : porte sur un service de type location, hébergement ou transport ; note de 1 à 5 ; commentaire facultatif.
- Seul l'auteur peut modifier ou supprimer ses sujets, réponses et avis.
- **Écrans** : Forum (onglets sujets et avis), détail d'un sujet, formulaires sujet et avis. La route `/forum/avis` ouvre directement l'onglet Avis.

---

## 3. Architecture

```
Écrans (screens/)  →  Repositories (*_repository.dart)  →  DatabaseHelper (singleton)  →  SQLite
                                  ↘ ApiService (HTTP, serveur IA) ↗
```

- **Écrans** : ils n'écrivent jamais en base directement. Ils passent par un repository.
- **Repositories** : une classe par module, avec les requêtes SQL et les règles métier (contrôles d'accès, transactions).
- **Modèles** : chaque modèle a `fromMap` (lecture) et `toMap` ou `toMapCreation`/`toMapSaisie` (écriture).
- **Schémas** : chaque module déclare ses `CREATE TABLE IF NOT EXISTS` dans `<module>_schema.dart`. `DatabaseHelper` les exécute toutes à la création et à chaque ouverture. La table `utilisateur` est créée en premier, à cause des clés étrangères.
- **Base** : `DatabaseHelper` est un singleton. À l'ouverture, `PRAGMA foreign_keys = ON` est activé. Les données de départ sont insérées une seule fois, à la création (voir §10).
- **Routes** : chaque module a un fichier `<module>_routes.dart` (chemins et fonction `pour(titre)`). `app_routes.dart` relie les chemins aux écrans, et `modules_routes.dart` relie le titre d'une tuile de l'accueil à sa route.
- **Session** : `SessionService` (`shared_preferences`) ne stocke que l'identifiant de l'utilisateur connecté.
- **Thème et composants** : `AppColors` (palette, jamais de couleur écrite en dur), `AppTheme` (Material 3, Plus Jakarta Sans via `google_fonts`), widgets communs dans `core/widgets/` (`ui_kit`, `travel_button`, `travel_field`, `travel_header`, `coming_soon_screen`).
- **Utilitaires** : `Validators` (saisie), `Montants` (montants en centimes), `dates` (formats), `PasswordHasher` (mots de passe).
- **Démarrage** (`main.dart`) : initialisation du binding Flutter, configuration de SQLite pour le web, ouverture de la base, puis `runApp`.
- **Web** : `sqflite_common_ffi_web` avec `sqlite3.wasm` et `sqflite_sw.js` (fichiers copiés dans `web/` par le script de version web).

---

## 4. Base de données

- Fichier : `travel_platform.db`. Version de schéma : `1`.
- Pas de migration : une modification de table ne s'applique pas à une base déjà créée (voir §13).
- Conventions de stockage :
  - dates : texte au format `AAAA-MM-JJ` ;
  - horodatages (`cree_le`) : texte ISO-8601 en UTC ;
  - montants : nombre réel en dinars, arrondi à 2 décimales ;
  - statuts : texte, contrôlé par `CHECK` quand la valeur est fixe.

| Table | Colonnes | Contraintes et clés étrangères |
|---|---|---|
| `utilisateur` | id, nom, prenom, email, telephone, role, mot_de_passe_hash, sel, cree_le | email `UNIQUE COLLATE NOCASE` ; role ∈ client, traveller, transporter, admin |
| `actualite` | id, titre, contenu, date_publication, auteur_id | auteur_id → utilisateur (SET NULL) |
| `voyage` | id, titre, destination, date_debut, date_fin, budget, statut, utilisateur_id | statut ∈ planifie, actif, termine, annule ; utilisateur_id → utilisateur (CASCADE) |
| `transporteur` | id, nom, type, contact | — |
| `reservation_transport` | id, depart, arrivee, date_depart, prix, statut, voyage_id, transporteur_id | `CHECK (depart <> arrivee)` ; voyage_id → voyage (CASCADE) ; transporteur_id → transporteur (SET NULL) |
| `commande_transport` | id, client_id, depart, arrivee, date_souhaitee, budget, description, statut, voyageur_id, cree_le | `CHECK (budget >= 0)` ; `CHECK (depart <> arrivee)` ; statut ∈ ouverte, acceptee, annulee ; client_id → utilisateur (CASCADE) ; voyageur_id → utilisateur (SET NULL) |
| `mise_en_relation` | id, commande_id, auteur_id, voyageur_id, transporteur_id, statut, cree_le | statut ∈ proposee, acceptee, refusee ; commande_id → commande_transport (CASCADE) ; auteur_id et voyageur_id → utilisateur (CASCADE) ; transporteur_id → transporteur (SET NULL) |
| `vehicule` | id, marque, modele, prix_par_jour, disponible | disponible vaut 1 par défaut |
| `location_vehicule` | id, date_debut, date_fin, prix_total, voyage_id, vehicule_id | voyage_id → voyage (CASCADE) ; vehicule_id → vehicule (CASCADE) |
| `hebergement` | id, nom, type, adresse, prix_par_nuit | — |
| `reservation_hebergement` | id, date_arrivee, date_depart, nb_personnes, prix_total, voyage_id, hebergement_id | nb_personnes vaut 1 par défaut ; voyage_id → voyage (CASCADE) ; hebergement_id → hebergement (CASCADE) |
| `forum_sujet` | id, titre, contenu, cree_le, auteur_id | auteur_id → utilisateur (CASCADE) |
| `forum_reponse` | id, contenu, cree_le, sujet_id, auteur_id | sujet_id → forum_sujet (CASCADE) ; auteur_id → utilisateur (CASCADE) |
| `avis` | id, cible_type, cible_id, note, commentaire, cree_le, auteur_id | `CHECK cible_type ∈ location, hebergement, transport` ; `CHECK note BETWEEN 1 AND 5` ; auteur_id → utilisateur (CASCADE) ; cible_id sans clé étrangère (cible polymorphe) |

Données insérées à la création de la base :
- un compte administrateur de test, seulement en mode debug (voir §10) ;
- deux actualités de bienvenue.

---

## 5. API REST et serveur IA

### 5.1 Client HTTP (`lib/core/services/api_service.dart`)

- Adresse de base : `http://10.0.2.2:3000/api`. L'adresse `10.0.2.2` désigne le PC depuis l'émulateur Android.
- Méthodes : `get`, `post`, `put`, `patch`, `delete`. Délai de 15 secondes.
- Une réponse hors 2xx lève une `ApiException` contenant le code et le corps de la réponse.
- Dans les fichiers examinés, seul le module transport utilise ce client, pour l'IA. Les autres modules fonctionnent en local.

### 5.2 Serveur IA (`server/`)

Routes (préfixe `/api/transport`) :

| Route | Rôle | Entrée | Sortie |
|---|---|---|---|
| `POST /interpreter-commande` | Phrase → champs de commande | `texte`, `date_reference` | `depart`, `arrivee`, `date`, `budget` |
| `POST /classer-voyageurs` | Classement des voyageurs | `depart`, `arrivee`, `date`, `candidats` | liste `{voyageur_id, score, raison}` triée |
| `POST /resumer-trajet` | Résumé d'un trajet | `depart`, `arrivee`, `date`, `prix`, `statut`, `transporteur` | `{resume}` |

- Lancement, configuration et tests : voir `server/README.md`.
- Port `3000`, écoute sur `127.0.0.1` par défaut.
- Codes HTTP : `200` succès, `400` entrée invalide, `404` route inconnue, `405` méthode refusée, `413` corps trop grand, `502` réponse du modèle invalide ou service indisponible, `503` clé API absente.
- L'application bascule sur ses règles locales à chaque erreur (`SourceIa.local`). L'écran de saisie affiche la source de la réponse (serveur ou local).

---

## 6. Intelligence artificielle (cahier des charges §7.2)

Règles du cahier des charges (§7.3) : appel au modèle uniquement depuis le serveur ; clé lue dans `ANTHROPIC_API_KEY` ; un prompt système par tâche ; sortie validée avant usage ; prix et accès calculés par l'application, jamais par le modèle ; aucun paramètre `temperature`, `top_p` ni `top_k`.

Modèle : `claude-haiku-5-5` (API Messages d'Anthropic).

| Module | Tâche | État |
|---|---|---|
| Transporteur (Abir Gamoudi) | Interprétation d'une commande en texte libre | Fait : serveur, repli local, écran Nouvelle commande |
| Transporteur (Abir Gamoudi) | Classement des voyageurs pour une commande | Fait : serveur, repli local, écran Commandes ouvertes |
| Transporteur (Abir Gamoudi) | Résumé de trajet | Fait : serveur, repli local, écran Voyageur |
| Voyage (Mouhamed Dhia Dridi) | Ébauche d'itinéraire jour par jour | Non trouvée dans les fichiers examinés |
| Location (Roua Dhafer) | Comparaison de véhicules en une phrase | Non trouvée dans les fichiers examinés |
| Hébergement (Oussema Mrayah) | Extraction de filtres de recherche | Non trouvée dans les fichiers examinés |
| User (Abdeslem Kessentini) | Brouillon d'actualité à valider | Non trouvée dans les fichiers examinés |
| Forum (Sirine Hamzaoui) | Résumé des avis (pour / contre) | Non trouvée dans les fichiers examinés |
| Forum (Sirine Hamzaoui) | Filtrage des messages (acceptable, spam, abusif) | Non trouvée dans les fichiers examinés |

« Non trouvée dans les fichiers examinés » signifie que la tâche n'apparaît ni dans les dépôts ni dans les écrans que j'ai relus. Les écrans des autres modules n'ont pas tous été relus : à confirmer.

Repli local du transport : interprétation par expressions régulières (« de X à Y », « demain », date jj/mm, budget en DT) ; classement par score (destination +50, date dans le voyage à 3 jours près +30, voyage à venir +20) ; résumé par gabarit de texte.

---

## 7. Installation (Windows 10 / 11)

Les quatre scripts `.bat` sont à la racine du projet. Ils lancent des scripts PowerShell et écrivent un journal dans `logs/`.

1. **`1_INSTALLER_ENVIRONNEMENT.bat`** (à lancer une fois par PC, en administrateur) : installe Git, Visual Studio Code et Android Studio avec winget ; clone Flutter (branche stable) dans `C:\src\flutter` ; ajoute Flutter au PATH utilisateur ; installe le SDK Android et l'extension VS Code Flutter. Ensuite, étapes manuelles dans Android Studio : assistant « Standard », *SDK Tools* → « Android SDK Command-line Tools », plugin Flutter, création d'un appareil virtuel (Pixel 7, API 34 ou plus), puis `flutter doctor --android-licenses` et `flutter doctor`.
2. **`2_PREPARER_PROJET.bat`** : accepte les licences Android ; crée `android/` et `ios/` si absents (`flutter create --org tn.esprit --project-name projet_mobile`) ; ajoute la permission `INTERNET` et `usesCleartextTraffic="true"` dans `AndroidManifest.xml` ; lance `flutter pub get`, `flutter analyze`, `flutter test` et `flutter doctor -v`. Le script ne modifie pas le code de `lib/`. Journal : `logs/projet.log`.
3. **`3_VERSION_WEB.bat`** : ajoute la plateforme web si `web/index.html` est absent, lance `pub get`, copie `sqlite3.wasm` et `sqflite_sw.js` dans `web/` (3 essais), puis analyse et tests. **Android Studio doit être fermé** pour éviter un fichier verrouillé. Journal : `logs/web.log`.
4. **`4_TESTER_TRANSPORT.bat`** : vérification rapide (`pub get`, `analyze`, `test`) avec un code de sortie pour chaque étape. Il utilise `C:\src\flutter\bin\flutter.bat` s'il existe, sinon le PATH. Journal : `logs/transport.log`. En cas de code différent de 0, envoyer ce fichier.

Les scripts `setup_windows.ps1`, `setup_projet.ps1` et `setup_web.ps1` ne doivent être lancés que par les `.bat` correspondants.

---

## 8. Lancer l'application

```powershell
flutter pub get
flutter run          # émulateur Android démarré (ou appareil connecté)
flutter test
```

- **Émulateur Android** : cas principal. Le module transport utilise l'IA si le serveur tourne (voir §5.2).
- **Version web** (après `3_VERSION_WEB.bat`) : `flutter run -d chrome`. Dans Chrome, l'adresse `10.0.2.2` n'est pas joignable : le module transport utilise alors ses règles locales.
- **Windows (application de bureau)** : non supportée. `main.dart` n'initialise pas `sqflite_common_ffi` pour le bureau, donc la base ne s'ouvre pas sous Windows. Les tests, eux, utilisent `sqflite_common_ffi` (dépendance de développement).
- **Serveur IA** : dans un second terminal, voir `server/README.md`.

---

## 9. Tests

### 9.1 Tests Flutter (`flutter test`)

| Fichier | Nombre | Contenu |
|---|---|---|
| `test/database_test.dart` | 5 | Inscription et session (bon rôle), e-mail unique sans casse, connexion (mauvais puis bon mot de passe), inscription admin impossible, profil modifiable et actualités de départ présentes |
| `test/unit_test.dart` | 5 | `Validators` : e-mail, mot de passe (8 caractères, une lettre, un chiffre), téléphone ; hachage salé (même mot de passe, sels différents, hachages différents) ; admin absent de l'inscription |
| `test/modules_test.dart` | 11 | Dates et montants (5) ; voyages CRUD, modification et suppression réservées au propriétaire ; location (prix, chevauchement, indisponibilité) ; hébergement (nuits, suppression en cascade) ; forum (seul l'auteur modifie) ; administration (actualités, comptes) |
| `test/transport_test.dart` | 10 | Règles de saisie (2) ; trajets et transporteurs CRUD, recherche, contrainte départ ≠ arrivée, suppression d'une fiche (3) ; commandes et mises en relation : coordonnées masquées puis visibles, droits du client, commande acceptée non supprimable (3) ; assistant IA en repli local (2) |

Total : **31 tests Flutter**.

### 9.2 Tests du serveur IA (`npm test` dans `server/`)

**13 tests** : validation des entrées, format de la requête vers Anthropic (modèle, en-têtes, absence de `temperature`, `top_p`, `top_k`), réponses avec JSON entouré de texte ou de balises, identifiants inconnus ignorés, scores bornés, noms et contacts non transmis au modèle, erreur amont sans fuite du corps, codes 400, 404, 405, 502 et 503. Les tests utilisent un faux service Anthropic local : aucun appel réseau réel.

### 9.3 Dernière exécution enregistrée

Le journal `logs/projet.log` (25/10/2026) indique `flutter analyze` et `flutter test` avec code de sortie 0. Ce journal date d'avant les dernières modifications du module transport, des écrans d'accueil et du serveur IA : **relancer `4_TESTER_TRANSPORT.bat` avant toute remise**.

---

## 10. Comptes et données de départ

- **Compte administrateur de test** : `admin@travel.tn` / `Admin1234`. Il n'existe qu'en mode debug (`kDebugMode`) et n'est jamais créé dans l'application de production.
- **Données de départ** : deux actualités de bienvenue. Leur auteur est le compte de test en mode debug, et il n'y a pas d'auteur en production.
- Aucun autre compte n'est fourni : utiliser l'inscription (rôles Client, Voyageur, Transporteur).
- Les données restent sur chaque appareil. GitHub ne partage que le code, pas la base.

---

## 11. Règles de saisie et de validation

| Champ | Règle |
|---|---|
| E-mail | Format standard ; unique, sans tenir compte de la casse |
| Mot de passe | 8 caractères minimum, au moins une lettre et un chiffre |
| Téléphone | 8 à 15 chiffres ; `+` et espaces acceptés (ex. `+216 20 123 456`) |
| Montant | Chiffres avec 0 à 2 décimales ; virgule ou point (`12,5` ou `12.50`) ; positif ou nul ; calcul en centimes entiers pour éviter les erreurs d'arrondi |
| Date | Saisie par sélecteur ; stockage `AAAA-MM-JJ` ; affichage « 15 OCT. 2026 » |
| Voyage | Date de fin ≥ date de début |
| Location | Pas de chevauchement avec une autre réservation du même véhicule ; véhicule disponible ; prix = tarif × jours (bornes comprises) |
| Hébergement | Date de départ strictement après la date d'arrivée ; prix = tarif × nuits |
| Transport | Départ ≠ arrivée ; date souhaitée non passée ; budget positif ou nul |
| Avis | Note entière de 1 à 5 |

---

## 12. Sécurité

- **Mots de passe** : stockés sous forme de `SHA-256(sel + mot de passe)`, avec un sel aléatoire de 16 octets par compte. **Limite** : SHA-256 est un algorithme rapide. Pour une mise en production, il faudrait un algorithme de dérivation de clé (PBKDF2, bcrypt ou argon2) côté serveur.
- **Session** : seul l'identifiant est stocké. Le rôle est relu en base, donc modifier les préférences ne suffit pas à changer de rôle.
- **Administrateur** : le rôle n'est pas proposé à l'inscription, et le compte administrateur ne peut pas être supprimé. Le compte de test n'existe qu'en mode debug.
- **Coordonnées** : téléphones visibles seulement après acceptation d'une commande ou d'une proposition.
- **Clé API** : `ANTHROPIC_API_KEY` est lue uniquement par le serveur IA. Elle n'est ni dans le code, ni dans l'application, ni dans le dépôt. Ne jamais committer de clé ni de fichier `.env`.
- **Serveur IA** : aucune authentification. Il écoute sur `127.0.0.1` par défaut. Ne pas l'exposer sur un réseau que vous ne contrôlez pas.
- **Réseau** : `usesCleartextTraffic="true"` autorise le HTTP en clair, pour le développement local uniquement. En production, utiliser HTTPS.

---

## 13. Dépannage et journaux

| Symptôme | Cause probable | Solution |
|---|---|---|
| « 'flutter' n'est pas reconnu » | Flutter absent du PATH dans la fenêtre ouverte | Rouvrir PowerShell ; vérifier `C:\src\flutter\bin` ; relancer le `.bat` |
| Fichier verrouillé pendant la configuration web | Android Studio ouvert | Fermer Android Studio, relancer `3_VERSION_WEB.bat` |
| Nouvelle colonne absente dans une table existante | Base déjà créée sans migration (une nouvelle table, elle, est créée automatiquement) | Désinstaller l'application de l'émulateur (la base est recréée, les données locales sont perdues), puis `flutter run` |
| Licences Android refusées | Licences non acceptées | `flutter doctor --android-licenses`, répondre `y` |
| Les appels IA ne répondent pas sur l'émulateur | Serveur arrêté, mauvais port ou pare-feu | Vérifier que `node index.js` tourne sur le port 3000 ; autoriser Node dans le pare-feu Windows |
| Comportement « local » dans Chrome | Attendu : `10.0.2.2` n'est pas joignable depuis le navigateur | Aucune action, sauf à changer l'adresse de base pour la version web |
| Analyse ou tests en échec | Erreur de code ou d'environnement | Envoyer `logs/transport.log` |

Journaux dans `logs/` :

| Fichier | Produit par | Contenu |
|---|---|---|
| `install.log` | `setup_windows.ps1` | Installation de l'environnement |
| `projet.log` | `setup_projet.ps1` | Préparation du projet, analyse, tests (dernière exécution enregistrée : 25/10/2026, code 0 pour l'analyse et les tests) |
| `web.log` | `setup_web.ps1` | Configuration de la version web |
| `transport.log` | `4_TESTER_TRANSPORT.bat` | Vérification du module transport (dernière exécution enregistrée : 28/10/2026, Flutter introuvable dans le PATH) |

---

## 14. Workflow d'équipe

Règles tirées de `CONTRIBUTING.md` :

- Chaque membre modifie **uniquement** le dossier de son module (`lib/modules/<module>/`) et son fichier `<module>_schema.dart`.
- `app_routes.dart` : chacun ajoute ses propres lignes de routes, rien d'autre.
- `lib/core/` et `pubspec.yaml` : modifiés par le responsable, après accord du groupe. Un nouveau package est ajouté par une seule personne.
- Branches : `main` ne reçoit pas de poussée directe ; une branche `feature/<membre>` par membre ; une Pull Request vers `main`, relue par un autre membre.
- Avant chaque poussée : `flutter analyze` (0 erreur) puis `flutter test`.
- Modification d'une table existante : désinstaller l'application de l'émulateur (voir §13).
- Conventions : fichiers en `snake_case.dart`, classes en `PascalCase` ; un modèle avec `toMap`/`fromMap` ; accès à la base uniquement par un repository ; `if (!mounted) return;` après chaque `await` dans un écran ; couleurs uniquement via `AppColors`.

Points à corriger dans la documentation d'équipe :
- `CONTRIBUTING.md` cite `lib/features/` et `exemple_crud/`, qui n'existent pas dans le projet actuel (le code est dans `lib/modules/`) ; il cite aussi une table `users`, alors que la table réelle est `utilisateur`.
- Le dossier `projet_mobile` examiné sur le PC ne contient pas de dépôt Git (pas de dossier `.git`) : la procédure de branches suppose un dépôt à initialiser ou à cloner.
- `docs/repartition_modules.md` est un modèle vide : les colonnes « Nom & prénom » et « Tables » sont à remplir.

### Ajouter votre module

1. Travaillez dans `lib/modules/<votre_module>/` sur la branche `feature/<module>`.
2. Déclarez vos tables dans `<module>_schema.dart` avec `CREATE TABLE IF NOT EXISTS`. Après une modification, désinstallez l'application de l'émulateur.
3. Créez le modèle (`fromMap`, `toMap`), le repository, les écrans, en réutilisant `core/widgets` et `AppColors`.
4. Ajoutez vos routes dans `<module>_routes.dart` (fonction `pour(titre)`) et enregistrez-les dans `modules_routes.dart` et `app_routes.dart`.
5. Dans `modules/user/screens/accueil_tab.dart`, remplacez l'écran « Module à venir » de vos tuiles par votre écran.
6. `flutter analyze` puis `flutter test` avant chaque poussée.

---

## 15. État d'avancement et points ouverts

### Fait

- **Interface** : écrans d'accueil (Splash, onboarding, connexion, inscription), accueil par rôle, profil, actualités, thème reconstruit à partir du UI Kit.
- **Données** : CRUD dans les six modules, avec les contraintes décrites en §4.
- **Vérification SQL** : contrôles exécutés hors Flutter sur le module transport (29 contrôles : CRUD, `CHECK`, clés étrangères, cascades, transaction d'acceptation, masquage des coordonnées). Ces contrôles n'ont pas été rejoués dans l'application.
- **IA transport** : trois tâches du §7.2, avec serveur, repli local et tests.

### Points ouverts

1. **Acceptation d'une commande (§5.3)** : aucune réservation de trajet n'est créée. Il faut décider du prix et de la date du trajet.
2. **Prix négatif** : `reservation_transport.prix` n'a pas de contrainte `CHECK (prix >= 0)`. Mesuré : SQLite accepte `-5`. Ajouter la contrainte impose une migration, car la version de schéma est `1` et il n'existe pas de `onUpgrade`.
3. **Autres tâches IA du §7.2** : non trouvées dans les fichiers examinés (voir §6).
4. **Écart avec le rapport** : le rapport décrit MySQL, une API REST avec JWT et un accès distant ; l'application utilise SQLite local, sans authentification serveur. Le serveur IA ne touche pas à la base.
5. **Serveur IA** : testé avec un faux service Anthropic uniquement. À tester avec une vraie clé ; aucune authentification.
6. **Mots de passe** : SHA-256 salé, insuffisant pour une production (voir §12).
7. **Version web** : l'adresse `10.0.2.2` n'est pas joignable depuis le navigateur.
8. **Windows (bureau)** : non supporté par `main.dart`.
9. **Dépendance web** : `sqflite_common_ffi_web: any` dans `pubspec.yaml` ; figer une version.
10. **Écarts avec le UI Kit** : nombre d'onglets de navigation (3 ou 4) à trancher ; Explorer et détail d'une destination notés comme non reproduits lors de la session précédente.
11. **Exécution à refaire** : `4_TESTER_TRANSPORT.bat` et `flutter run` après les dernières modifications.
12. **Documentation d'équipe** : voir §14.

---

## 16. Structure du projet

Arborescence du dossier au moment de la rédaction (dossiers `android/`, `ios/` et `build/` omis, générés par Flutter).

```
projet_mobile/
├── assets/
│   └── images/
│       ├── kit_logo.png
│       ├── onboarding_1.jpg
│       ├── onboarding_2.jpg
│       └── onboarding_3.jpg
├── docs/
│   └── repartition_modules.md
├── lib/
│   ├── core/
│   │   ├── database/
│   │   │   ├── database_helper.dart
│   │   │   ├── web_setup_stub.dart
│   │   │   └── web_setup_web.dart
│   │   ├── routes/
│   │   │   ├── app_routes.dart
│   │   │   └── modules_routes.dart
│   │   ├── services/
│   │   │   ├── api_service.dart
│   │   │   └── session_service.dart
│   │   ├── theme/
│   │   │   ├── app_colors.dart
│   │   │   └── app_theme.dart
│   │   ├── utils/
│   │   │   ├── dates.dart
│   │   │   ├── montants.dart
│   │   │   ├── password_hasher.dart
│   │   │   └── validators.dart
│   │   └── widgets/
│   │       ├── coming_soon_screen.dart
│   │       ├── travel_button.dart
│   │       ├── travel_field.dart
│   │       ├── travel_header.dart
│   │       └── ui_kit.dart
│   ├── modules/
│   │   ├── forum/
│   │   │   ├── models/
│   │   │   │   ├── avis.dart
│   │   │   │   └── sujet_forum.dart
│   │   │   ├── screens/
│   │   │   │   ├── avis_form_screen.dart
│   │   │   │   ├── avis_tab.dart
│   │   │   │   ├── forum_screen.dart
│   │   │   │   ├── sujet_detail_screen.dart
│   │   │   │   ├── sujet_form_screen.dart
│   │   │   │   └── sujets_tab.dart
│   │   │   ├── forum_repository.dart
│   │   │   ├── forum_routes.dart
│   │   │   └── forum_schema.dart
│   │   ├── hebergement/
│   │   │   ├── models/
│   │   │   │   ├── hebergement.dart
│   │   │   │   └── reservation_hebergement.dart
│   │   │   ├── screens/
│   │   │   │   ├── gestion_etablissements_screen.dart
│   │   │   │   ├── hebergement_form_screen.dart
│   │   │   │   ├── hebergement_screen.dart
│   │   │   │   ├── hebergements_tab.dart
│   │   │   │   ├── sejour_form_screen.dart
│   │   │   │   └── sejours_tab.dart
│   │   │   ├── hebergement_repository.dart
│   │   │   ├── hebergement_routes.dart
│   │   │   └── hebergement_schema.dart
│   │   ├── location/
│   │   │   ├── models/
│   │   │   │   ├── location_vehicule.dart
│   │   │   │   └── vehicule.dart
│   │   │   ├── screens/
│   │   │   │   ├── gestion_flotte_screen.dart
│   │   │   │   ├── location_form_screen.dart
│   │   │   │   ├── location_screen.dart
│   │   │   │   ├── locations_tab.dart
│   │   │   │   ├── vehicule_form_screen.dart
│   │   │   │   └── vehicules_tab.dart
│   │   │   ├── location_repository.dart
│   │   │   ├── location_routes.dart
│   │   │   └── location_schema.dart
│   │   ├── transporteur/
│   │   │   ├── models/
│   │   │   │   ├── commande_transport.dart
│   │   │   │   ├── mise_en_relation.dart
│   │   │   │   ├── reservation_transport.dart
│   │   │   │   ├── transporteur.dart
│   │   │   │   └── voyageur_candidat.dart
│   │   │   ├── screens/
│   │   │   │   ├── commande_form_screen.dart
│   │   │   │   ├── commandes_ouvertes_screen.dart
│   │   │   │   ├── mes_commandes_screen.dart
│   │   │   │   ├── relations_screen.dart
│   │   │   │   ├── reservations_screen.dart
│   │   │   │   ├── trajet_form_screen.dart
│   │   │   │   ├── transporteur_form_screen.dart
│   │   │   │   └── voyageur_screen.dart
│   │   │   ├── transport_format.dart
│   │   │   ├── transport_ia.dart
│   │   │   ├── transport_repository.dart
│   │   │   ├── transport_routes.dart
│   │   │   ├── transport_widgets.dart
│   │   │   └── transporteur_schema.dart
│   │   ├── user/
│   │   │   ├── screens/
│   │   │   │   ├── accueil_tab.dart
│   │   │   │   ├── actualite_form_screen.dart
│   │   │   │   ├── actualites_tab.dart
│   │   │   │   ├── garde_admin.dart
│   │   │   │   ├── gestion_actualites_screen.dart
│   │   │   │   ├── home_shell.dart
│   │   │   │   ├── login_screen.dart
│   │   │   │   ├── onboarding_screen.dart
│   │   │   │   ├── profil_tab.dart
│   │   │   │   ├── signup_screen.dart
│   │   │   │   ├── splash_screen.dart
│   │   │   │   ├── utilisateurs_screen.dart
│   │   │   │   └── widgets_accueil.dart
│   │   │   ├── actualite.dart
│   │   │   ├── admin_repository.dart
│   │   │   ├── admin_routes.dart
│   │   │   ├── dashboard_repository.dart
│   │   │   ├── role.dart
│   │   │   ├── user_repository.dart
│   │   │   ├── user_schema.dart
│   │   │   └── utilisateur.dart
│   │   └── voyage/
│   │       ├── models/
│   │       │   └── voyage.dart
│   │       ├── screens/
│   │       │   ├── voyage_form_screen.dart
│   │       │   └── voyages_screen.dart
│   │       ├── voyage_repository.dart
│   │       ├── voyage_routes.dart
│   │       └── voyage_schema.dart
│   ├── app.dart
│   └── main.dart
├── logs/
│   ├── install.log
│   ├── projet.log
│   ├── transport.log
│   └── web.log
├── server/
│   ├── test/
│   │   └── transport_ai.test.js
│   ├── anthropic.js
│   ├── index.js
│   ├── package.json
│   ├── README.md
│   └── transport_ai.js
├── test/
│   ├── database_test.dart
│   ├── modules_test.dart
│   ├── transport_test.dart
│   └── unit_test.dart
├── web/
│   ├── icons/
│   │   ├── Icon-192.png
│   │   ├── Icon-512.png
│   │   ├── Icon-maskable-192.png
│   │   └── Icon-maskable-512.png
│   ├── favicon.png
│   ├── index.html
│   ├── manifest.json
│   ├── sqflite_sw.js
│   └── sqlite3.wasm
├── .gitignore
├── .metadata
├── 1_INSTALLER_ENVIRONNEMENT.bat
├── 2_PREPARER_PROJET.bat
├── 3_VERSION_WEB.bat
├── 4_TESTER_TRANSPORT.bat
├── analysis_options.yaml
├── CONTRIBUTING.md
├── devtools_options.yaml
├── projet_mobile.iml
├── pubspec.lock
├── pubspec.yaml
├── README.md
├── setup_projet.ps1
├── setup_web.ps1
└── setup_windows.ps1
```
