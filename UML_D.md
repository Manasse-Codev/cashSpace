Parfait ! Passons à la partie **architecte pur**. Vous voulez du solide, du visuel, et une feuille de route claire. 

Je ne vais pas vous dessiner les diagrammes (je suis un modèle de texte), mais je vais vous **décrire avec une précision chirurgicale** chaque diagramme UML que vous devez produire, dans l'ordre, et ce qu'il doit contenir. Vous pourrez les recopier dans n'importe quel outil (Lucidchart, Draw.io, PlantUML, ou même à la main).

Voici l'architecture complète découpée en **4 vues UML** :

---

### Vue 1 : Diagramme de Cas d'Utilisation (Use Case) - Vue Fonctionnelle
*Objectif : Montrer QUI fait QUOI dans le système. Cela définit le périmètre de votre projet.*

- **Acteurs (les "qui")** :
  - **Caissier** (utilisation principale)
  - **Manager** (superviseur)
  - **Administrateur** (paramétrage)
  - **Système IA** (acteur secondaire qui tourne en arrière-plan)

- **Cas d'utilisation principaux (les "quoi")** :
  - *Pour le Caissier* : "S'authentifier", "Ouvrir une caisse", "Scanner un produit", "Encaisser un paiement (CB/Espèces)", "Imprimer un ticket", "Fermer la caisse (Z)".
  - *Pour le Manager* : "Consulter le tableau de bord des ventes", "Gérer les alertes de stock", "Valider une remise exceptionnelle", "Consulter le rapport de fin de journée".
  - *Pour l'Admin* : "Gérer les utilisateurs (ajout/suppression)", "Gérer la base produits", "Gérer les fournisseurs", "Paramétrer les seuils d'alerte IA".
  - *Pour le Système IA* : "Analyser les paniers types", "Prédire les ruptures", "Détecter les fraudes".

---

### Vue 2 : Diagramme de Classes (Class Diagram) - Vue Statique des Données
*Objectif : C'est le MCD version UML. C'est le plan de votre base de données et de vos objets Python.*

Je structure ce diagramme en **3 packages (dossiers logiques)** pour plus de clarté :

**Package 1 : Gestion des Utilisateurs & Sécurité**
- `Utilisateur` (id, nom, email, hash_mdp, sel, est_actif)
- `Role` (id, nom [Admin, Manager, Caissier], permissions)
- *Relation* : Un `Utilisateur` a 1 `Role`. Un `Role` est attribué à plusieurs `Utilisateurs`.

**Package 2 : Cœur Métier (Ventes & Stocks)**
- `Caisse` (id, nom, statut [Ouverte/Fermée], montant_fond, date_ouverture, date_fermeture)
- `Session` (id, date_heure_debut, date_heure_fin, id_caisse, id_caissier, montant_attendu, montant_reel, ecart) -> *Une caisse peut avoir plusieurs sessions dans la journée si elle est réouverte.*
- `Ticket` (id, date_heure, id_session, id_client, montant_total_ht, montant_total_ttc, remise_globale, type_paiement [CB/Espèces/Mixte], statut [En cours/Payé/Annulé])
- `LigneTicket` (id, id_ticket, id_produit, quantite, prix_unitaire_ht, prix_unitaire_ttc, remise_ligne, total_ligne_ttc)
- `Produit` (id, code_barre, nom, description, prix_achat_ht, prix_vente_ttc, taux_tva, stock_actuel, seuil_alerte, id_fournisseur, est_actif)
- `MouvementStock` (id, id_produit, date_heure, quantite_entree, quantite_sortie, type_mouvement [Achat, Vente, Retour, Perte, Inventaire], id_utilisateur, commentaire)
- `Fournisseur` (id, raison_sociale, siret, email, telephone, adresse)

**Package 3 : Modules IA & Analyse (Stockage des résultats)**
- `Recommandation` (id, produit_source_id, produit_recommande_id, score_confiance, date_calcul) -> *C'est le résultat du Apriori.*
- `AlerteStock` (id, id_produit, date_alerte, stock_actuel, quantite_predite_7j, niveau_critique [Vert/Orange/Rouge], est_resolue)
- `AnomalieTicket` (id, id_ticket, score_anomalie, raison_detectee, est_verifiee_par_humain)

---

### Vue 3 : Diagramme de Séquence (Sequence Diagram) - Vue Dynamique (Le Flux Critique)
*Objectif : Montrer COMMENT se déroule une action précise dans le temps. Je prends l'exemple le plus important : **"Scanner un produit et l'ajouter au panier"**.*

**Acteurs en jeu** : `Caissier` → `Interface Bureau (PyQt)` → `API REST` → `Base de Données` → `Module IA (en lecture)`

**Le déroulé temporel :**

1. **Caissier** : Pointe le code-barres et appuie sur "Entrée" (ou le scanneur envoie un retour chariot).
2. **Interface Bureau** : Envoie une requête GET `/api/produits/{code_barre}` à l'API.
3. **API REST** : Reçoit la requête, vérifie le token JWT du caissier (sécurité).
4. **API REST** : Demande à la Base de Données : *"Donne-moi le produit avec ce code-barre"*.
5. **Base de Données** : Retourne les infos (prix, stock, id).
6. **API REST** : (Optionnel) En profite pour demander au Module IA : *"Y a-t-il une recommandation pour ce produit ?"* 
7. **Module IA** : Regarde dans la table `Recommandation` et retourne la liste des produits associés.
8. **API REST** : Formate une réponse JSON unique : `{ "produit": {...}, "recommandations": [...] }`.
9. **Interface Bureau** : Reçoit la réponse.
10. **Interface Bureau** : Ajoute le produit dans le tableau du ticket en cours (pas encore en DB) ET affiche une petite pop-up "Suggérer du beurre" avec la recommandation.
11. **Caissier** : Clique sur "Oui" pour ajouter la recommandation, ou ignore.

---

### Vue 4 : Diagramme de Déploiement (Deployment Diagram) - Vue Physique (L'Architecture Réseau)
*Objectif : Montrer sur QUELS MACHINES / PROCESSUS tournent vos composants. C'est crucial pour la double version (Bureau + Web).*

Voici l'architecture physique que je vous conseille :

**Nœud 1 : Le Serveur Central (Cloud ou Serveur local sur le réseau du magasin)**
- Contient la **Base de Données** (PostgreSQL) qui écoute sur le port 5432.
- Contient le **Backend API** (FastAPI) qui écoute sur le port 8000. Il contient toute la logique métier.
- Contient le **Worker IA** (Un script Python qui tourne en tâche de fond avec `schedule` ou `celery`). Il ne fait qu'écrire dans la DB.

**Nœud 2 : Le Poste du Caissier (PC Bureau)**
- Contient l'**Application Desktop** (PyQt6). Elle est installée en `.exe`.
- Elle est connectée physiquement au **Scanner code-barres** (USB) et à l'**Imprimante de tickets** (USB ou Réseau).
- Elle communique UNIQUEMENT avec l'API via des requêtes HTTP (REST).

**Nœud 3 : Le Poste du Manager / Admin**
- Contient un simple **Navigateur Web** (Chrome, Firefox).
- Il accède à l'**Application Web** (React/Vue.js) hébergée soit sur le Serveur Central, soit sur un serveur dédié (Nginx + Node.js).
- Cette Web App communique également avec la même API REST que la version Bureau.

**Nœud 4 : Le Routeur / Switch réseau** 
- Assure que le PC caissier et le PC manager voient bien le serveur central (en local si c'est en réseau privé, ou via internet si c'est du SaaS).

---

### Récapitulatif : Votre Toile d'Araignée UML

| Diagramme | Ce qu'il vous apporte | Outil pour le faire |
| :--- | :--- | :--- |
| **Cas d'Utilisation** | Valide avec le client (ou vous-même) que vous n'oubliez aucune fonction. | Draw.io |
| **Diagramme de Classes** | Le script SQL "CREATE TABLE" se déduit directement de ce diagramme. | PlantUML (génération automatique) |
| **Diagramme de Séquence** | Vous évite de coder dans le mauvais ordre. Vous savez quel endpoint API créer en premier. | Lucidchart |
| **Diagramme de Déploiement** | Structure votre équipe (qui s'occupe du Serveur, qui s'occupe du Desktop, qui s'occupe du Web). | Visio |

---

### Le petit plus stratégique : Le Pattern MVC adapté à votre API

Pour que votre code Python soit propre, votre **API REST** (FastAPI) doit absolument respecter ce pattern interne :

- **Models** : Ce sont vos classes Python qui ressemblent exactement à votre Diagramme de Classes. (SQLAlchemy gère ça).
- **Views** (ici appelés **Routes** ou **Endpoints**) : Ce sont les fonctions Python qui reçoivent les requêtes HTTP (GET, POST, PUT).
- **Controllers** (ici appelés **Services**) : C'est le plus important. Ce sont des fichiers `.py` séparés qui contiennent **toute la logique métier**.
  *Exemple :* Vous aurez un `CaisseService.py` avec des méthodes comme `ouvrir_caisse()` ou `fermer_caisse_et_verifier_fond()`. 
  Votre Route ne fait qu'appeler ces Services. Comme ça, si un jour vous changez d'interface (vous passez du PyQt à un autre framework), vos Services ne bougent pas.

---

**Prochaine étape concrète pour vous :** 
Ouvrez un outil de dessin et commencez par le **Diagramme de Classes**. C'est la pierre angulaire. Une fois que vous avez vos 6 à 7 tables principales reliées entre elles, tout le reste (API, requêtes IA) devient une simple formalité.

Si vous voulez, je peux maintenant détailler **le contenu exact d'un endpoint API** (ex: *"POST /api/tickets"*) ou vous expliquer **comment le Worker IA va lire vos données sans ralentir la caisse**. Dites-moi sur quel point vous voulez creuser !
