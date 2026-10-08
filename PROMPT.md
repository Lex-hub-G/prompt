# Mission : adaptation du pipeline de prévision 

## 1. Contexte


Deux cas d'usage existent :
- **Souscription** : prévision du burning cost par entreprise et grand poste .
- **Macro / consommation portefeuille** : prévision du burning cost par sous-poste et cluster démographique, afin d'estimer l'inflation  et d'alimenter un outil Excel .

Ma mission concerne le deuxième cas d'usage, plus précisément **le délégataire le gestionnaire délégué**.

Un pipeline complet existe déjà pour la **gestion directe**. Mon objectif est de l'adapter aux données le gestionnaire délégué, en conservant dans un premier temps les clusters, les hyperparamètres et la méthodologie de modélisation utilisés sur le Direct.

Je souhaite que tu m'accompagnes dans cette adaptation de manière rigoureuse, en analysant le code existant plutôt qu'en reconstruisant un pipeline différent.

## 2. Environnement et données

L'environnement de travail est **Databricks**, avec des notebooks Python, des tables de données et des modèles NeuralProphet.

Les notebooks existants se trouvent dans l'environnement France, dans un répertoire proche de :

`[chemin interne des notebooks à renseigner]`

Les dernières versions utilisées sont notamment identifiées par `[identifiant de version à vérifier]`.

Les données le gestionnaire délégué sont déjà préparées dans le **Data Model** de Databricks, avec une table par grand poste, selon une convention de nommage proche de :

`[table de segmentation par grand poste]`

Ces tables contiennent notamment :
- Année et mois de consommation.
- Grand poste et sous-poste médical.
- Sexe, tranche d'âge, région et type de bénéficiaire.
- TP/HTP (tiers payant / hors tiers payant).
- Expositions mensuelles.
- Remboursements assureur.
- Nombre de sinistres.

Les noms et chemins exacts devront être vérifiés dans l'environnement. Ne les invente pas.

Le **burning cost** est défini par :

BC = Remboursements assureur / Exposition

Les données ont un arrêté comptable prévu au **31 août 2026**.

## 3. Objectif immédiat : scénario 1 le gestionnaire délégué

Je dois produire un premier scénario de prévision en respectant les contraintes suivantes :

1. **Réutiliser le clustering du Direct**, sans recalculer les clusters.
2. **Réutiliser les hyperparamètres NeuralProphet optimisés sur le Direct**, sans relancer de grid search.
3. Utiliser les données segmentées le gestionnaire délégué comme nouvelles données d'entrée.
4. Appliquer les corrections de consommation par les **PSI/IBNR**, avec les tables de cadence appropriées.
5. Appliquer **deux mois de censure** aux derniers mois observés.
6. Commencer l'entraînement en **janvier 2021**, sous réserve de disponibilité effective des données le gestionnaire délégué.
7. Avec un arrêté fin août 2026 et deux mois censurés, utiliser normalement **fin juin 2026** comme fin d'entraînement.
8. Produire des prévisions jusqu'au **31 décembre 2027**.
9. Conserver les régresseurs du scénario de référence et vérifier la disponibilité de leurs valeurs futures.
10. Fixer une **random seed** pour assurer la reproductibilité.
11. Enregistrer les résultats dans des tables propres à le gestionnaire délégué, sans écraser les tables du Direct.
12. Produire la table finale permettant d'alimenter l'Excel de la Direction Technique.

## 4. Notebook 1 : pré-processing

Analyse le notebook de préparation des données existant et identifie précisément les modifications nécessaires.

### A. Clustering

Le notebook charge les définitions de clusters du Direct, dans des tables proches de `[table des affectations de clusters]`.

Chaque groupe démographique et chaque sous-poste sont associés à un cluster.

Je dois conserver cette correspondance et l'appliquer aux données le gestionnaire délégué.

Vérifie que les variables de segmentation, leurs types et leurs valeurs sont compatibles entre les deux sources.

### B. Chargement le gestionnaire délégué

Remplace les références aux tables de consommation segmentées Direct par les tables le gestionnaire délégué correspondantes.

Vérifie les noms de colonnes, les formats de dates et la disponibilité des données.

### C. Corrections PSI/IBNR

Analyse le chargement et l'application des tables de cadence.

Vérifie particulièrement :
- Le calcul du lag.
- La date d'arrêté comptable.
- Les facteurs de développement appliqués.
- La pertinence des cadences pour le gestionnaire délégué.
- Les éventuels facteurs manquants ou incohérents.

Ne suppose pas que les cadences du Direct sont automatiquement valables pour le gestionnaire délégué.

### D. Agrégation

Les expositions peuvent être répétées sur plusieurs lignes correspondant à différents sous-postes ou modalités TP/HTP.

Assure-toi que :
- Les remboursements sont correctement additionnés.
- Les expositions ne sont pas comptées plusieurs fois.
- Les regroupements démographiques respectent les définitions des clusters.
- Les burning costs sont correctement calculés.

### E. Sortie

Produis une table finale contenant les séries mensuelles par grand poste, sous-poste et cluster, avec les expositions, remboursements observés, remboursements corrigés et burning costs.

Conserve autant que possible le format attendu par le notebook suivant.

## 5. Notebook 2 : forecasting NeuralProphet

Analyse le notebook existant et adapte-le pour le gestionnaire délégué.

### A. Hyperparamètres

Charge les meilleurs hyperparamètres issus de l'optimisation sur le Direct, dans la table de type `[table des meilleurs hyperparamètres]`.

Vérifie la correspondance entre les sous-postes, les clusters et les paramètres disponibles.

Préserve les éventuelles exceptions métier déjà présentes dans le code, notamment pour l'orthodontie, en les signalant explicitement.

### B. Régresseurs externes

Analyse la table des régresseurs et leur utilisation effective.

Parmi les variables potentiellement disponibles :
- PMSS.
- ONDAM total.
- ONDAM soins de ville.
- MonPsy.
- Nombre de samedis par mois.
- Taux de remboursement dentaire.
- Tiers payant partiel.
- Autres changements réglementaires.

Ne suppose pas que tous ces régresseurs sont actifs dans tous les modèles.

Pour le scénario 1, conserve la configuration de référence et identifie les éventuelles incompatibilités avec le gestionnaire délégué.

Vérifie également les valeurs futures nécessaires jusqu'à décembre 2027, notamment lorsqu'elles reposent sur des hypothèses de la Direction Technique.

### C. Censure et période d'entraînement

Applique deux mois de censure.

Les derniers mois censurés peuvent avoir des consommations observées corrigées par PSI, mais ne doivent pas être utilisés comme observations d'entraînement.

Vérifie précisément la logique de sélection des dates et les éventuels décalages de lag.

### D. Entraînement et prévision

Lance NeuralProphet pour chaque série correspondant à un cluster et à un sous-poste.

Conserve les paramètres de reproductibilité et les conventions du pipeline existant.

Produis les prédictions jusqu'à décembre 2027 et vérifie qu'aucun segment attendu n'a été oublié.

## 6. Résultats et Excel

Reprends les traitements existants pour combiner historique et prévisions.

La table finale doit contenir notamment :
- Grand poste et sous-poste.
- Mois.
- Exposition / nombre de membres.
- Montant remboursé.
- Burning cost observé.
- Burning cost prédit.
- Identifiant du scénario.

Vérifie les agrégations des clusters, en utilisant les expositions appropriées plutôt qu'une moyenne simple non pondérée des burning costs.

Adapte les tables `[table de sortie pour Excel]` et les références aux scénarios pour le gestionnaire délégué.

Ne modifie pas les résultats des scénarios du Direct.

Le résultat doit être compatible avec l'Excel existant.

## 7. Vérifications et validation

Avant de considérer le scénario terminé, effectue autant que possible les vérifications suivantes :

- Cohérence des dates et du périmètre le gestionnaire délégué.
- Absence de doublons inattendus.
- Conservation des expositions et remboursements lors des agrégations.
- Couverture des groupes démographiques par le clustering Direct.
- Cohérence des corrections PSI/IBNR.
- Absence de fuite temporelle entre entraînement et prévision.
- Disponibilité des hyperparamètres et régresseurs.
- Absence de valeurs manquantes ou de prévisions aberrantes.
- Cohérence des prévisions aux niveaux cluster, sous-poste et grand poste.
- Compatibilité de la table finale avec l'Excel.

Ne présente pas comme validées les étapes qui n'ont pas réellement été exécutées.

## 8. Méthode de collaboration attendue

Je veux travailler **étape par étape**, et non recevoir immédiatement une réécriture complète du pipeline.

Commence par examiner les notebooks et les tables que je te fournirai.

Pour chaque notebook :

1. Explique son rôle dans le pipeline global.
2. Identifie les données d'entrée et de sortie.
3. Repère les parties spécifiques au Direct.
4. Indique précisément les modifications nécessaires pour le gestionnaire délégué.
5. Propose les changements de code minimaux.
6. Explique les conséquences techniques et statistiques de ces modifications.
7. Indique les vérifications à effectuer avant de poursuivre.

**Contraintes importantes :**
- Ne réécris pas inutilement du code qui fonctionne.
- Ne modifie pas la méthodologie sans justification.
- Ne crée pas de noms de tables, colonnes ou chemins fictifs.
- Ne supprime pas les règles métier existantes sans les comprendre.
- Distingue ce qui est confirmé par le code de ce qui reste une hypothèse.
- Signale les risques de biais, de fuite temporelle et d'erreur d'agrégation.
- Privilégie du code clair, reproductible et compatible avec Databricks.

## 9. Travaux ultérieurs — hors scénario 1

Une fois le premier scénario terminé, nous pourrons étudier :
- L'impact du tiers payant partiel spécifique à le gestionnaire délégué.
- Des censures différentes selon les postes, notamment quatre mois en hospitalisation.
- Un régresseur pour les revalorisations hospitalières de 2026.
- De nouveaux scénarios réglementaires.
- Un backtesting temporel plus robuste.
- La comparaison des estimations PSI et des prédictions NeuralProphet.
- La transférabilité des hyperparamètres du Direct vers le gestionnaire délégué.

Ces sujets ne doivent pas bloquer la réalisation du premier scénario.

## 10. Première action attendue

Commence par me demander le **premier notebook de pré-processing** ou par l'examiner s'il est déjà disponible dans ton environnement.

Présente-moi ensuite :
- Son architecture.
- Les tables qu'il lit et produit.
- Les sections à conserver.
- Les sections à modifier pour le gestionnaire délégué.
- Les éventuels points à vérifier avant exécution.

Nous avancerons ensuite cellule par cellule jusqu'à obtenir un scénario 1 le gestionnaire délégué fonctionnel.
