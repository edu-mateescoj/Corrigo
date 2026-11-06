# Audit d'intégrité des copies — baseline (6 novembre 2026)

> Branche de travail : `audit/initial`. Ce document est un relevé de lecture du **code Java** ; ce n'est pas une certification du logiciel. Aucun jeu de copies scannées n'a encore été exécuté de bout en bout.

## Objectif prioritaire

**Aucune page, annotation ou note de la copie A ne doit pouvoir être attribuée silencieusement à la copie B.**

Distinguer :
- identifiant technique immuable de la copie (généré et vérifiable) ;
- identifiant manuscrit pseudonyme de l'élève (donnée à reconnaître, jamais preuve unique) ;
- rang de la copie dans le scan (information de préparation, jamais clé d'attribution définitive).

## Observations vérifiées dans le code (branche `master`, baseline `69408b8a`)

| Référence | Observation vérifiée | Risque / statut |
|---|---|---|
| `NewEvaluationPlan.java:65-72` | Les plages de pages sont dérivées de `i * pagesPerCopy`, avec une taille constante. | Aucune vérification intrinsèque de l'identité réelle des pages ; une insertion/suppression peut décaler les correspondances. |
| `NewEvaluationPlan.java:81-92` | Les pages supplémentaires donnent `EXTRA_PAGES`, **non bloquant**. | Le code autorise un import avec des pages non affectées. Le test `NewEvaluationPlanTest.cutsTheScanRegularly` confirme ce comportement. |
| `NewEvaluationPlan.java:83-86` | La détection des doublons se fait sur les noms de **fichiers de sortie**. | Avec préfixes numériques, deux pseudonymes identiques peuvent donner des noms de fichiers différents ; pas de validation de l'unicité de l'identifiant manuscrit. |
| `NewEvaluationWizard.java:307-355` | L'assistant affiche une vignette du haut de la première page de chaque copie. | Contrôle humain partiel ; pas de validation du contenu de toutes les pages. |
| `NewEvaluationWizard.java:423-478` | L'extraction utilise `PageExtractor(document, firstPage + 1, lastPage + 1)`. | Découpage par plages de pages PDF ; **pas de désimposition A3→A4 dans ce chemin d'import**. |
| `ExerciseCorrectionWorkflow.java:18-20` | La navigation vers la page d'exercice borne l'indice à la dernière page existante. | Une carte de pages incohérente peut conduire visuellement à une mauvaise page sans erreur explicite ; cela ne démontre pas un transfert de note. |
| `EvaluationFolders.java:199-219` | Les sauvegardes de données d'évaluation écrivent un fichier temporaire puis le déplacent, avec solution de repli si le déplacement atomique est indisponible. | Point favorable pour l'intégrité d'écriture ; reste à tester les coupures et les changements de copie. |
| `Edition.java:560-587` | Le fichier d'annotations YAML est retrouvé via le chemin/nom du PDF (local ou centralisé), et non par une empreinte du contenu. | Remplacer manuellement le PDF à chemin identique peut réassocier des annotations préexistantes à un autre contenu. Cas à reproduire en test ; ce n'est pas une preuve de confusion en usage normal. |
| `FairnessExport.java:19-25,39-61` | Export Markdown anonymisé des corrections, désignées `Copie 01`, etc., et des erreurs/méthodes récurrentes. | Opportunité d'analyse qualitative ; prévoir une table de correspondance vérifiable et stable hors de l'export. |

## Ce que les tests réussis démontrent — et ne démontrent pas

Sur Windows, à partir de `master` au commit `69408b8a`, l'utilisateur a exécuté `gradlew.bat clean test --no-daemon --console=plain` : **BUILD SUCCESSFUL**. Nombre de tests, couverture et tests d'intégration du scan collectif non encore relevés. Les tests unitaires existants ne suffisent donc pas à certifier l'absence de mélange des copies.

## Contrats prioritaires à formaliser et tester

**P0 — Avant toute correction réelle**
1. Le scan original est conservé et identifié par SHA-256.
2. Chaque page source a exactement une destination attendue (ou une exclusion manuelle explicite et consignée).
3. Toute anomalie de comptage, doublon d'identifiant, page manquante/insérée ou discordance de bornes bloque l'import jusqu'à validation explicite.
4. Chaque PDF élève porte un identifiant technique immuable ; un manifeste enregistre `source_hash`, indices des pages PDF source, ordre des pages A4, identifiant élève vérifié, hash du PDF produit.
5. Les livrets A3 imposés sont désassemblés dans une étape explicite, testée sur différentes orientations de scan.
6. Les notes et annotations sont écrites, relues et exportées via l'identité du document ; permutation d'ordre, renommage, fermeture/réouverture et interruption ne doivent jamais transférer de notes vers une autre copie.

**P1 — Correction qualitative assistée**
7. Les observations IA citent une zone source (`copy_id`, page et rectangle) et une catégorie d'erreur révisable.
8. Aucune reconnaissance/OCR/IA ne modifie une note définitive sans validation humaine.
9. L'export de synthèse des erreurs se recoupe avec les annotations et les notes individuelles ; les fréquences sont reproductibles.

## Jeux d'essai indispensables

- 3 copies factices, 4 pages A4 chacune, avec identifiant visible sur toutes les pages ; numérotation des pages et motifs graphiques uniques.
- Scans A3 recto verso réguliers, avec inversion des faces ou rotation d'un côté.
- Copies inversées, identifiant dupliqué, page manquante, page en trop, doublon de page, feuille mêlant deux élèves.
- Navigation aléatoire entre copies, annotation de l'exercice 2 de B, retour à A, sauvegarde/relecture, export comparé au manifeste.
- Réponses hors zones attendues : l'analyse IA doit déclarer l'incertitude plutôt qu'inventer une réponse absente.

## Points en attente d'audit

- Contrôle des fichiers d'édition par copie (classe `Edition`) au changement de document et au redémarrage.
- Validation concrète des exports CSV/PDF/Moodle et cohérence des identifiants.
- Vérification de l'interaction des vues d'exercices, du recalcul des notes et des tags/erreurs fréquentes.
- Résultats GitHub Actions sur la branche d'audit et couverture des tests.

Aucune modification du fonctionnement n'est autorisée implicitement par ce document : la correction des points P0 doit se faire avec des tests de régression et validation explicite.
