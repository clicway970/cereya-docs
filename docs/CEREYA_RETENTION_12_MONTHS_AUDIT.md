# Cereya — audit transversal de rétention à 12 mois

5 octobre 2026. Préparation locale uniquement. Aucun accès production, aucune purge de données utilisateur, aucun push/déploiement/tag/cron installé.

## Inventaire vérifié

| Produit | Dépôt | FR / DE | CGV avant | Privacy avant | Purge complète avant |
|---|---|---|---|---|---|
| HPI adulte — historique + Cognitive V0 autonome | `cereya` | Oui / oui | Non chiffrée | FR/DE : cible 12 mois, exceptions vagues ; analytics 24 mois | Non identifiée |
| HPI enfant | `cereya-hpi-enfant` | Oui / oui | Non chiffrée | FR/DE : cible 12 mois ; FR analytics 24 mois | Non identifiée |
| TDAH adulte | `cereya-tdah` | Oui / oui | Non chiffrée | FR/DE : cible 12 mois ; analytics 24 mois | Non identifiée |
| TDAH enfant | `cereya-tdah-enfant` | Oui / oui | Non chiffrée | FR : cible 12 mois / analytics 24 mois ; DE : durée non précisée | Non identifiée |
| TSA adulte — historique + natif | `cereya-tsa` | Oui / oui | Non chiffrée | FR : cible 12 mois / analytics 24 mois ; DE : finalités, aucune durée chiffrée | Non identifiée |
| TSA enfant | `cereya-tsa-enfant` | Oui / oui | Non chiffrée | FR : cible 12 mois ; DE : maximum 12 mois avec exceptions ; analytics 24 mois | Non identifiée |
| AuDHD adulte | `cereya-audhd-adulte` | Oui / oui | Premium 24m, reprise 12m | FR/DE : brut/reprise 12 mois depuis création ; snapshot/accès payé 24 mois depuis paiement | Partielle brute 12m / payé 24m |

`cereya-vitrine` : site institutionnel et CMS (articles, pages, références, utilisateurs administrateurs), aucun dossier d’évaluation identifié ; aucune modification. `cereya-cognitive-core` : bibliothèque scientifique et démo avec transport local, sans Prisma ni service de persistance utilisateur ni dépôt Git ; aucun changement. `cereya-docs` : documentation, pas un produit d’évaluation. Le parcours Cognitive V0 actif est couvert dans `cereya`, pas traité comme une huitième base inventée.

## Matrice cible et réserves

| Produit | CGV 12m | Privacy 12m | Purge réelle | Dry-run | A–F PostgreSQL FR/DE | Cron préparé |
|---|---|---|---|---|---|---|
| HPI adulte — historique + Cognitive V0 autonome | PASS local | PASS local | PASS local | PASS | PASS | Chemins FR/DE documentés, installation externe |
| HPI enfant | PASS local | PASS local | PASS local | PASS | PASS | CLI prête ; emplacement FR/DE à confirmer |
| TDAH adulte | PASS local | PASS local | PASS local | PASS | PASS | CLI prête ; emplacement DE à confirmer |
| TDAH enfant | PASS local | PASS local | PASS local | PASS | PASS | CLI prête ; emplacement DE à confirmer |
| TSA adulte — historique + natif | PASS local | PASS local | PASS local | PASS | PASS | CLI prête ; emplacement DE à confirmer |
| TSA enfant | PASS local | PASS local | PASS local | PASS | PASS | CLI prête ; emplacement DE à confirmer |
| AuDHD adulte | PASS local | PASS local | PASS local | PASS | PASS | CLI prête ; emplacement FR/DE à confirmer |

Les sept dépôts de code ont la même interface opérateur, avec leur propre inventaire de relations et leurs migrations propres. Voir dans chacun `docs/evaluation-retention-12-months.md`. Les pages publiques contrôlées sont `/cgv`, `/politique-confidentialite` (FR), `/agb`, `/datenschutz` (DE).

## Règle et commandes

Finalisée : `completed_at` (Evaluation, TsaEvaluation) ou `finalized_at` (CognitiveV0Session). Inachevée : `created_at`. Échéance : ajout de **12 mois calendaires**, UTC, jour de fin de mois borné (29/02 → 28/02). Ni paiement, ni relance, ni modification administrative ne repoussent cette échéance. La création est retenue pour les abandons : un simple accès ou une action BO ne permet pas une conservation indéfinie.

Depuis le checkout et l’environnement de **la bonne instance** :

```sh
npm run retention:purge
npm run retention:purge -- --dry-run --batch-size=50
npm run retention:purge -- --execute --batch-size=50 --max-batches=100
```

Sans option : dry-run. `--execute` doit être explicite ; `--execute --dry-run` et les options inconnues sont refusés. Un dry-run examine au maximum 50 dossiers par racine (jusqu’à 500 avec `--batch-size`) et compte les anciens journaux ; il ne crée rien, même temporairement, et sa transaction est READ ONLY. Execute traite plusieurs lots jusqu’à épuisement ou limite de sécurité. Code de sortie 0 succès, 1 échec, 2 limite de lots atteinte. Ne pas interpréter le nombre d’un seul dry-run comme le volume total d’un rattrapage massif.

`runRetention` dans `scripts/evaluation-retention.mjs` travaille en transaction SERIALIZABLE, avec verrou advisory par base et verrou des dossiers. Paiement concurrent : même verrou du dossier pour AuDHD ; les conflits ou échecs font échouer/annuler le lot sans archive partielle. Relancer un lot est idempotent. Les lots limitent les transactions, pas le nombre de réponses d’un dossier. Le graphe de FK de `evaluation-retention-config.json` provient du schéma réel de ce dépôt ; une relation DB non inventoriée entraîne un refus, avant toute suppression. Aucun endpoint public n’est ajouté.

Une demande d’effacement anticipé reste possible. Après vérification proportionnée de l’identité et du dossier, l’opérateur peut utiliser la même chaîne transactionnelle :

```sh
npm run retention:purge -- --execute --root=evaluations --erase-id=UUID_DU_DOSSIER '--confirmation=EFFACER UUID_DU_DOSSIER'
```

Adapter `--root` à `cognitive_v0_sessions` ou `tsa_evaluations` pour les parcours natifs présents. L’UUID et le tableau racine sont validés ; la confirmation exacte est obligatoire en execute. Ce lot ne lance aucune demande utilisateur réelle et n’ajoute pas de bouton BO.

## Suppressions et archives

Suppression physique du dossier et des objets rattachés : questionnaire, exercices, temps/traces, résultats/scoring persisté, snapshots/rapports, accès/grants/tokens, commandes opérationnelles et leurs snapshots de contrat, factures en base, relances, emails et événements techniques rattachés. Les tables de questions/exercices et leurs contenus scientifiques restent intacts. Les commandes payées ou remboursées sont copiées **avant suppression** dans `evaluation_accounting_archive`, sans FK vers le dossier.

Liste fermée de l’archive : UUID commande, statut déjà persisté, montants HT/TVA/TTC, devise, date commande/paiement/remboursement, numéro/date/annulation de facture et référence de paiement si présente, date d’archivage. Aucun email, identifiant d’évaluation, catégorie HPI/TDAH/TSA/AuDHD, réponse, score, rapport, token, snapshot JSON ni motif manuel. La TVA manquante reste NULL : le modèle `TsaOrder` stocke le TTC et un sous-total commercial, pas une ventilation fiscale fiable. Ne pas déduire une TVA du sous-total. Les éventuels éléments fiscaux externes doivent être rapprochés par le responsable comptable.

L’archive minimale ne remplace pas une chaîne de facturation complète. Les factures/documents comptables externes, leur durée légale et leurs accès restreints restent à valider avec le responsable comptable. Aucun effacement comptable légal n’est lancé à 12 mois. Les données d’évaluation ne sont pas conservées sous prétexte de comptabilité.

Les événements analytics individuels, email hashes, journaux email/sécurité/audit âgés de 12 mois sont purgés indépendamment des dossiers. Les événements associés au dossier expiré sont supprimés même s’ils sont récents ; les références explicites dans JSON/URL sont nettoyées. Un email/hash seul ne démontre pas le rattachement : les journaux sans provenance explicite suivent leur propre limite de 12 mois depuis création, avec réserve à auditer pour les anciennes lignes détachées. Les `ActiveVisit` sont limités à 24 heures et leurs liens personnels effacés avec le dossier. `AnalyticsSnapshot` n’est pas supprimé : il n’est conservable que pour des métriques véritablement agrégées/anonymes (recette : compteurs sans identifiants). Aucune anonymisation d’un hash n’est prétendue. Les runs TSA de relances stockent des compteurs et motifs de saut agrégés, sans dossier.

Newsletter : consentement indépendant conservé, lien vers Evaluation et source d’évaluation supprimés. Aucune conservation de l’email du dossier au titre d’un hypothétique usage futur. Promotions personnelles : adresse/détails/rattachement supprimés, code remplacé et désactivé ; un squelette de coupon peut rester pour ne pas casser une autre commande encore active. Les demandes RGPD traitées sont nettoyées après 12 mois depuis traitement ; une demande encore en cours n’est pas abandonnée automatiquement.

## Planification et monitoring

Une exécution nocturne **quotidienne** par base suffit. La purge ne dépend pas de l’ouverture du BO. Les logs JSON contiennent uniquement produit, date, mode, compteurs et état ; pas d’email, réponse, score ou token, pas de message Prisma susceptible de contenir un paramètre. Le shell du cron doit capturer stdout/stderr et le code de sortie. Journal à faire tourner selon l’exploitation existante. Le contrôle BO existant « Crons & sauvegardes » lit `SYSTEM_CRON_LOG_PATH_EVALUATION_RETENTION`, affiche date/compteurs/état, et signale erreurs, dry-runs et exécution vieille de plus de 36 h. Surveiller quotidiennement absence d’exécution, `retention_failed`, `batch_limit_reached` et backlog ; aucun succès de cron VPS n’est revendiqué ici.

Pour un chemin d’instance encore inconnu : se placer sur le VPS dans le checkout actif identifié via PM2/Nginx, vérifier son remote Git, puis **afficher sans installer** la ligne portant son vrai chemin :

```sh
DEPLOY_DIR=$(pwd -P)
printf '0 3 * * * cd "%s" && npm run retention:purge -- --execute >> "%s/retention-purge.log" 2>&1\n' "$DEPLOY_DIR" "$DEPLOY_DIR"
```

Ce générateur n’établit pas qu’un checkout est en production : vérifier d’abord processus, locale, environnement et base, et ne pas ajouter deux jobs pour la même base. Il évite d’inventer des chemins DE ou de reprendre un manuel copié d’un autre produit. Aucun cron n’a été installé dans ce lot.

## Preuves locales et limites

Migrations depuis une base neuve et Prisma generate : PASS. `test:retention` : PASS sur PostgreSQL 16 local, database URL explicitement fournie et limitée à `localhost/retention_test_*`. A : 11 mois 29 jours conservé, y compris création plus ancienne et finalisation récente. B : >12 mois supprimé, y compris abandon avec updatedAt récent. C : payé expiré, données supprimées, archive blanche seule conservée. D : deuxième exécution sans erreur. E : comparaison intégrale des tables avant/après dry-run, aucune écriture. F : chaque racine native et locales FR/DE. Vérifications supplémentaires : rollback après échec imposé, newsletter indépendante décorrélée, agrégats anonymes conservés, plafonnement du 29 février. Les fixtures complexes HPI/TSA sont insérées dans ces seules bases jetables avec triggers USER suspendus **dans la transaction de création des fixtures**, puis restaurés avant toute purge ; FK/CHECK restent actifs. La purge n’utilise jamais ce contournement.

Lint et TypeScript : PASS. CGV et privacy réellement rendues localement : 4 routes HTTP 200 par dépôt (FR et DE), règle et suppression automatique visibles. Serveurs loopback avec répertoires Next isolés ; fichiers générés restaurés. Ces vérifications ne sont pas des preuves de production ni une certification juridique. Aucun push, tag, déploiement ou connexion DB distante.

Réserves externes obligatoires : déployer/apppliquer les migrations et planifier le cron manuellement ; traiter sauvegardes, dumps/exports, Brevo/Stripe et autres sous-traitants selon leurs contrats et procédures ; une copie déjà téléchargée ne peut pas être effacée à distance. Le code courant génère les PDF à la demande, sans `pdfPath` ni écriture de fichier utilisée dans `src/server` ; les colonnes historiques `pdf_path` et d’éventuels fichiers externes doivent être inventoriés avant purge de dossiers réels, pour éviter un fichier orphelin. Aucun fichier utilisateur n’a été examiné ou supprimé.

La limite contractuelle est 12 mois ; la suppression physique intervient au passage du job quotidien (délai technique jusqu’au prochain passage). Les nouveaux contrôles natifs TSA/HPI finalisé/AuDHD bornent aussi l’accès à l’échéance ; les parcours historiques ont déjà une expiration de reprise/rapport depuis création, qui peut être plus tôt que la finalisation. Ne pas supprimer ces bornes antérieures pour prolonger un accès. Vérifier les lignes historiques sans expiration lors du dry-run VPS. Sauvegardes et sous-traitants restent un périmètre distinct : cette recette locale ne suffit pas à déclarer une conformité RGPD globale.

## Différences à conserver visibles

- HPI V0 : DELETE historiquement interdit ; nouvelle autorisation de suppression scoped dans une transaction privée, guards INSERT/UPDATE inchangés. Dépendances RESTRICT, vérifications, verbal, estimate récupéré, commandes/paiements/grants/reminders sont effectivement effacés. Une session récente liée à un vieux dossier historique est décorrélée, sans être supprimée avant sa propre échéance.
- TSA natif : effacement parent et cascade existante des données figées ; aucune protection scientifique désactivée. TTC archive exact ; HT/TVA NULL lorsqu’absents du modèle, à rapprocher de la facturation externe.
- TSA/TDAH enfant : ShadowEngineExecution et snapshots supprimés avec Evaluation ; TSA enfant : grants manuels supprimés également, sans fabriquer de commande ni de revenu.
- AuDHD : ancienne offre 24 mois explicitement remplacée par 12 mois depuis finalisation, version légale v4. Réserve contractuelle avant déploiement pour les anciens achats ; migration cap des accès, code de paiement, accès web/PDF, CGV/privacy/FAQ/emails et purge alignés. Ancienne archive AuDHD minimale conservée ; nouvelle archive commune indépendante utilisée par CLI.

## Cron : chemins réellement documentés, pas des attestations VPS

Source : `cereya/docs/analytics/Cereya_Funnel_Production_Extraction_v1.md`, manuels 13, `cereya-tsa/ecosystem.config.js`, `cereya-tsa-enfant/docs/09-architecture-technique.md` et réserve explicite AuDHD doc 12.

| Produit | FR | DE |
|---|---|---|
| HPI adulte — historique + Cognitive V0 autonome | `/opt/cereya` | `/opt/cereya-de` |
| HPI enfant | Réserve : chemin non établi | Réserve : chemin non établi |
| TDAH adulte | `/opt/cereya-tdah` | Réserve : chemin non établi |
| TDAH enfant | `/opt/cereya-tdah-enfant` | Réserve : chemin non établi |
| TSA adulte — historique + natif | `/opt/cereya-tsa` | Réserve : chemin non établi |
| TSA enfant | `/opt/cereya-tsa-enfant` | Réserve : chemin non établi |
| AuDHD adulte | Réserve : chemin non établi | Réserve : chemin non établi |

Lignes exactes pour les chemins versionnés (vérifier checkout, instance et DB sur VPS avant installation) :

```cron
0 3 * * * cd /opt/cereya && npm run retention:purge -- --execute >> /opt/cereya/retention-purge.log 2>&1
0 3 * * * cd /opt/cereya-de && npm run retention:purge -- --execute >> /opt/cereya-de/retention-purge.log 2>&1
0 3 * * * cd /opt/cereya-tdah && npm run retention:purge -- --execute >> /opt/cereya-tdah/retention-purge.log 2>&1
0 3 * * * cd /opt/cereya-tdah-enfant && npm run retention:purge -- --execute >> /opt/cereya-tdah-enfant/retention-purge.log 2>&1
0 3 * * * cd /opt/cereya-tsa && npm run retention:purge -- --execute >> /opt/cereya-tsa/retention-purge.log 2>&1
0 3 * * * cd /opt/cereya-tsa-enfant && npm run retention:purge -- --execute >> /opt/cereya-tsa-enfant/retention-purge.log 2>&1
```

Pour les autres instances, le générateur du paragraphe planification construit la ligne avec le chemin actif vérifié ; aucun suffixe DE n’est inventé. Le manuel HPI enfant répète celui d’HPI adulte ; le manuel TSA enfant contient aussi un chemin TDAH enfant hérité. AuDHD possède seulement des affectations préparatoires.

## Vérifications enregistrées

- Migrations et Prisma generate sur sept bases PostgreSQL neuves : PASS. Deux bases supplémentaires neuves pour AuDHD Premium FR/DE. Aucun environnement applicatif existant purgé.
- 7 suites `test:retention`, 2 tests DB déclarés chacune + 1 test de monitoring, scénarios A–F pour chaque langue et racine native : PASS (plus contrôles agrégats, rollback et refus de DELETE indépendant HPI).
- AuDHD Premium : 7 tests FR + 7 DE : PASS.
- Lint + TypeScript : PASS dans les sept dépôts.
- 28 pages CGV/privacy rendues HTTP 200, contenu visible 12m et automatique : PASS.
- Tests scoring des sept produits, Option C/v3 TSA, domaine TSA enfant : PASS. Parités AuDHD : 33 TDAH, 686 TSA natifs et 686 TSA v3.1 : PASS. Les fonctions scientifiques, seuils, banques et interactions ne sont pas modifiés.
- Suite unitaire AuDHD : le contrôle email a été mis à jour pour l’échéance 12m depuis finalisation ; environnement FR explicite requis pour la fixture i18n. PASS : 140/140 avec configuration FR explicite, test de monitoring inclus ; monitoring également validé dans chaque dépôt.
- Aucun build production n’est revendiqué : ce lot a vérifié types, lint, recettes PostgreSQL, tests scientifiques et le rendu Next local FR/DE.

## Checklist manuelle VPS avant activation

1. Identifier chaque processus et son checkout actif, FR/DE, domaines et base. Ne pas réutiliser un chemin issu d’un manuel copié.
2. Faire valider la transition des anciennes CGV AuDHD 24m avant migration/plafonnement/effacement réel.
3. Vérifier les lignes historiques (completion absente, liens expirés, factures/fichiers externes), l’archive comptable et les obligations applicables avec le responsable compétent.
4. Préparer et déployer manuellement les commits de chaque instance ; appliquer les migrations correspondantes et générer Prisma. Aucun de ces gestes n’a été réalisé ici.
5. Dans la bonne instance, lancer d’abord `npm run retention:purge -- --dry-run`, examiner les compteurs et estimer le nombre de lots.
6. Configurer `SYSTEM_CRON_LOG_PATH_EVALUATION_RETENTION` pour le BO. Planifier une première exécution autorisée, puis le job quotidien ; vérifier logs, code retour et backlog. Ne pas doubler une même base sous plusieurs jobs.
7. Valider rotation/restauration des sauvegardes et suppression des exports/dumps/PDF historiques : éviter la réintroduction de dossiers expirés après restauration.
8. Traiter Brevo, Stripe, sous-traitants et éventuels exports externes selon procédures/obligations ; le lot ne s’y connecte pas.
9. Contrôler chaque semaine dernière exécution et erreur ; faire valider le registre de conservation et la conformité juridique globale.

## Références de cadre

La séparation des finalités et de l’archivage est documentée par la [CNIL](https://www.cnil.fr/fr/passer-laction/les-durees-de-conservation-des-donnees) et son [cadre d’archivage intermédiaire](https://www.cnil.fr/fr/comment-concilier-les-durees-de-conservation-et-les-archives). L’[article L123-22 du Code de commerce](https://www.legifrance.gouv.fr/loda/article_lc/LEGIARTI000006219327/2021-06-23) porte sur la conservation des documents comptables et pièces justificatives ; il ne justifie pas la conservation de réponses ou rapports cognitifs. Ces références ne constituent pas une validation professionnelle des CGV ou de la chaîne de facturation.
