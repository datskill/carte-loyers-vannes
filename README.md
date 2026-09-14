# Loyers &amp; cadre de vie — Vannes et 25 km

## Contenu
- `index.html` — l'application (carte Leaflet + liste + fiche commune), sans dépendance à builder
- `data.js` — toutes les données (36 communes autour de Vannes), format JSON dans une variable JS
- `README.md` — ce fichier

## Utilisation
Ouvrez `index.html` dans un navigateur (double-clic). Une connexion internet est nécessaire uniquement pour
charger Leaflet, le fond de carte et la police (CDN) ; les données sont embarquées dans `data.js`.

## Interface
Trois colonnes : à gauche la liste des communes (recherche + 7 tris) et la sélection en cours, au centre la
carte, à droite la fiche de la commune active.

- **Colorer la carte** : bascule entre le loyer €/m² du type de bien choisi et le score de cadre de vie.
  La taille des cercles est proportionnelle à la population, le contour doré marque une commune sélectionnée.
- **Survol synchronisé** : survoler une ligne de la liste met en avant le point sur la carte, et inversement.
- **Fiche commune** : 3 indicateurs clés puis 4 onglets — Loyers, Cadre de vie, Sécurité, Transports.
- **Sélection** : l'étoile de la fiche ajoute la commune à la liste des communes à visiter (conservée dans le
  `localStorage` du navigateur). Le bouton « Comparer » ouvre un tableau trié colonne par colonne.

## Transports (emplacement prêt, données à venir)
L'onglet Transports d'une commune se remplit automatiquement dès qu'une clé `transports` est ajoutée à cette
commune dans `data.js` ; sans cette clé, un encart explique le format attendu :

```js
transports: {
  note: "Réseau Kicéo…",
  gares:  [{ nom, type, frequence, temps_paris_min }],
  lignes: [{ code, nom, reseau, type: "bus|car|train", frequence, couleur }]
}
```

## Données
Les loyers proviennent des fichiers ANIL/DHUP fournis par l'utilisateur :
- `pred-mai-mef-dhup.csv` → Maison (surface de référence 92 m²)
- `pred-app-mef-dhup.csv` → Appartement, tous types (52 m²)
- `pred-app3-mef-dhup.csv` → Appartement T3 et plus (72 m²)
- `pred-app12-mef-dhup.csv` → Appartement T1-T2 (37 m²)

Chaque commune est identifiée par son code INSEE. Les valeurs `loyer_m2` sont les loyers d'annonce prédits en
€/m² charges comprises, `nb_observations` est le nombre d'annonces ayant servi à l'estimation, `r2` est le
coefficient de qualité du modèle (R² ajusté). Un indicateur est marqué "fiable" si nb_observations >= 30 et
R² >= 0,5 — sinon il doit être interprété avec prudence (petites communes, faible volume d'annonces).

## Mettre à jour les données de loyer
Pour rafraîchir avec un millésime plus récent de l'ANIL, remplacez les 4 CSV sources et relancez le script de
génération (mapping nom de fichier -> type de bien, extraction par code INSEE, recalcul du loyer mensuel
estimé = loyer_m2 * surface de référence).

## Qualité de vie (score par commune)
Chaque commune porte un champ `qualite_vie` (score global 0-100 + détail par catégorie : Commerces, Santé &
social, Éducation, Sport/loisirs/culture, Services du quotidien, Transports), construit à partir de la
**Base Permanente des Équipements (BPE) 2024** de l'INSEE — recensement public et gratuit de tous les
commerces et équipements de France, géolocalisés par commune (https://www.insee.fr, dataset republié en
Parquet sur data.gouv.fr : https://www.data.gouv.fr/datasets/base-permanente-des-equipements-3).

**Méthodologie** (voir aussi `DATA.qualite_vie_meta` dans `data.js`) : pour chaque catégorie, le score
combine (a) la diversité des types d'équipements présents dans la commune rapportée à la diversité observée
sur l'ensemble des 36 communes (65 % du score), et (b) la densité d'équipements pour 1000 habitants,
normalisée sur une échelle logarithmique plafonnée au 95e centile observé, pour éviter qu'une très petite
commune avec un seul commerce n'obtienne un score artificiellement élevé (35 % du score). Le score global est
une moyenne pondérée des 6 catégories (Commerces 28 %, Santé & social 22 %, Éducation 18 %, Sport/loisirs/
culture 17 %, Services du quotidien 10 %, Transports 5 %). Le domaine "Tourisme" (hébergement touristique) de
la BPE est exclu, car peu pertinent pour la qualité de vie d'un résident.

## Sécurité (délinquance enregistrée)
Chaque commune porte aussi un champ `securite` (taux global + détail par catégorie et par indicateur),
construit à partir des **bases statistiques communales du SSMSI** (Ministère de l'Intérieur) —
https://www.data.gouv.fr/datasets/bases-statistiques-communale-departementale-et-regionale-de-la-delinquance-enregistree-par-la-police-et-la-gendarmerie-nationales.

**5 catégories** (voir `DATA.securite_meta`) regroupant 15 indicateurs SSMSI : Cambriolages, Vols (véhicule,
dans véhicule, accessoires, sans violence, avec/sans arme), Violences (physiques hors/dans cadre familial,
sexuelles), Stupéfiants (usage, trafic), Escroqueries & dégradations. Les taux sont des **moyennes
annuelles 2023-2025** (pour 1000 habitants), calculées uniquement sur les années où l'indicateur est publié
pour la commune.

**Limite importante à connaître** : pour les communes de moins d'environ 3500-5000 habitants, une partie des
15 indicateurs n'est **pas publiée** par le SSMSI (secret statistique lié aux faibles effectifs), et cela
varie d'un indicateur à l'autre au sein d'une même commune. Chaque commune affiche donc son nombre
d'indicateurs publiés sur 15 — un taux bas ou une mention "non publié" ne signifie **pas** que la commune est
plus sûre, cela peut simplement refléter une non-publication. Ces chiffres mesurent des faits enregistrés par
les forces de l'ordre (pas la délinquance réelle) et peuvent être influencés par la présence d'un
commissariat/brigade sur la commune. Pour cette raison, ces données ne sont volontairement **pas** intégrées
au score "Qualité de vie" ni à la coloration de la carte — elles sont affichées à titre informatif dans la
fiche de chaque commune.

Pour rafraîchir ces scores avec un millésime BPE plus récent :
1. Récupérer l'URL du fichier Parquet BPE à jour sur data.gouv.fr.
2. Interroger ce fichier à distance avec DuckDB (`INSTALL httpfs; LOAD httpfs;` puis
   `SELECT ... FROM read_parquet('<url>') WHERE DEPCOM IN (...)`) pour ne récupérer que les lignes des 36
   communes — inutile de télécharger le fichier national complet (~180 Mo), la requête filtrée ne transfère
   que quelques centaines de Ko grâce au filtrage par groupes de lignes Parquet.
3. Agréger par commune et domaine (1ère lettre de `TYPEQU`), recalculer les scores, et fusionner le résultat
   dans le champ `qualite_vie` de chaque commune dans `data.js`.
