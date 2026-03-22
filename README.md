# 🚦 ETL / Data Warehouse — Accidents Routiers France
### Projet BI pour le CRASR — Centre de Recherche sur les Accidents et la Sécurité Routière

---

## 📋 Contexte

Le CRASR centralise des données hétérogènes sur les accidents de la route en France 
pour produire des rapports self-service. L'objectif est de construire un datawarehouse 
permettant d'analyser les accidents à deux niveaux de granularité différents.

---

## ❓ Besoins métier — Questions auxquelles le DW doit répondre

### Analyse des accidents (niveau macro)
- Quelle est l'évolution du nombre d'accidents par année, région, département, ville ?
- Quelles sont les périodes les plus accidentogènes (mois, saison, heure de la journée) ?
- Quels types de routes sont les plus mortelles (nationale, départementale, autoroute) ?
- Combien de morts et blessés recensés par an ?
- Les conditions météo et l'état de la surface influencent-ils la gravité des accidents ?

### Analyse des véhicules et victimes (niveau micro)
- Y a-t-il une différence d'accidentologie entre femmes et hommes ?
- Le port de la ceinture / du casque réduit-il la gravité des blessures ?
- Quelle tranche d'âge des conducteurs est la plus impliquée dans des accidents mortels ?
- Quelles manœuvres sont les plus dangereuses ?
- Quel motif de déplacement est le plus accidentogène ?
- Quelle place dans le véhicule est la plus mortelle ?
- Les radars fixes ont-ils un impact sur le nombre de décès ?

---

## 🏗️ Pourquoi 2 tables de faits ?

Les questions métier se posent à **deux niveaux de granularité différents** :

| Niveau | Grain | Table de fait |
|---|---|---|
| **Macro** | 1 ligne = 1 accident | FactAccident |
| **Micro** | 1 ligne = 1 véhicule + 1 victime | FactVehicule |

Utiliser une seule table de faits aurait été impossible car :
- FactAccident agrège les victimes (nb_tues, nb_blesses...) → on perd le détail individuel
- FactVehicule garde le détail de chaque victime → trop fin pour analyser les accidents globalement

---

## 📐 Architecture — Star Schema
```
                    DimDate          DimTemps
                       ↘               ↙
DimLocalisation ←── FactAccident ──→ DimConditionsAccident
                       ↗               ↖
                  DimTopographie    DimRadar

DimConducteur ←──────────────────────────────→ DimTypeVehicule
                     FactVehicule
DimProfilVictime ←───────────────────────────→ DimCollision
```

---

## 📦 Sources de données

| Source | Format | Contenu |
|---|---|---|
| RelationalAccident.bak | SQL Server backup | Accidents 2005-2016 |
| france2016.txt | TSV INSEE | 39 837 communes |
| depts2016.txt | TSV INSEE | 101 départements |
| reg2016.txt | TSV INSEE | 18 régions |
| radars.xlsx | Excel | 3 343 radars fixes France |

---

## 🗂️ Dimensions — Justification et questions répondues

### DimDate — SCD0 — 11 323 lignes
**Pourquoi ?** Analyser la temporalité des accidents.  
**Questions :**
- Quelle année a eu le plus d'accidents ?
- Les accidents sont-ils plus fréquents le weekend ?
- Quelle saison est la plus dangereuse ?

### DimTemps — SCD0 — 1 440 lignes
**Pourquoi ?** Analyser l'heure exacte des accidents.  
**Questions :**
- À quelle heure de la journée les accidents sont-ils les plus fréquents ?
- Les heures de rush sont-elles plus dangereuses ?

### DimLocalisation — SCD2 — 35 885 lignes
**Pourquoi ?** Localiser géographiquement chaque accident. SCD2 car la réforme 
territoriale 2016 a créé de nouvelles régions.  
**Questions :**
- Quels départements/régions ont le plus d'accidents ?
- Y a-t-il des zones géographiques particulièrement dangereuses ?

### DimConditionsAccident — SCD1 — 600 lignes
**Pourquoi ?** Analyser l'impact des conditions environnementales.  
**Questions :**
- La pluie augmente-t-elle le nombre d'accidents mortels ?
- Les accidents de nuit sans éclairage sont-ils plus graves ?

### DimTopographie — SCD1 — 6 000 lignes
**Pourquoi ?** Analyser l'impact du type de route et d'intersection.  
**Questions :**
- Les routes hors agglomération sont-elles plus mortelles ?
- Quel type d'intersection est le plus dangereux ?

### DimTypeVehicule — SCD1 — 32 lignes
**Pourquoi ?** Analyser l'accidentologie par type de véhicule.  
**Questions :**
- Les 2-roues sont-ils surreprésentés dans les accidents mortels ?
- Les poids lourds causent-ils plus de décès ?

### DimCollision — SCD1 — 29 750 lignes
**Pourquoi ?** Analyser les circonstances précises de la collision.  
**Questions :**
- Quelles manœuvres sont les plus dangereuses ?
- Les obstacles fixes (arbres, glissières) causent-ils plus de morts ?

### DimConducteur — SCD2 — 30 lignes
**Pourquoi ?** Analyser le profil du conducteur impliqué. SCD2 car les 
tranches d'âge peuvent être redéfinies.  
**Questions :**
- Quelle tranche d'âge a le plus d'accidents mortels ?
- Les hommes ont-ils plus d'accidents que les femmes ?

### DimProfilVictime — SCD2 — 52 500 lignes
**Pourquoi ?** Analyser le profil détaillé de chaque victime. SCD2 car 
la classification de gravité peut évoluer.  
**Questions :**
- Le port de la ceinture réduit-il la gravité ?
- Quelle place dans le véhicule est la plus mortelle ?
- Quel motif de déplacement est le plus risqué ?

### DimRadar — SCD2 — 3 343 lignes
**Pourquoi ?** Analyser l'impact des radars sur la sécurité routière. 
SCD2 car la vitesse limite peut changer par décret.  
**Questions :**
- Les zones avec radars ont-elles moins d'accidents mortels ?
- Quel type de radar est le plus efficace ?

---

## 📊 Tables de faits — Mesures

### FactAccident — 252 288 lignes
| Mesure | Description |
|---|---|
| nb_vehicules_impliques | Nombre de véhicules impliqués dans l'accident |
| nb_victimes_total | Nombre total de victimes |
| nb_tues | Nombre de tués |
| nb_blesses_graves | Nombre de blessés graves |
| nb_blesses_legers | Nombre de blessés légers |
| nb_indemnes | Nombre de personnes indemnes |

### FactVehicule — 1 897 340 lignes
| Mesure | Description |
|---|---|
| nb_passagers | Nombre de passagers dans le véhicule |
| nb_pietons_percutes | Nombre de piétons percutés |

---

## 🔄 ETL SSIS — Architecture

### Ordre de chargement
```
DimDate → DimTemps → DimLocalisation → DimConditionsAccident
→ DimTopographie → DimTypeVehicule → DimCollision
→ DimConducteur → DimProfilVictime → DimRadar
→ FactAccident → FactVehicule
```

### Packages SSIS
| Package | Rôle | Type |
|---|---|---|
| MasterETL.dtsx | Orchestration complète séquentielle | Master |
| DimDate.dtsx | Génération 2000-2030 en T-SQL | SCD0 |
| DimTemps.dtsx | Génération 00:00-23:59 en T-SQL | SCD0 |
| DimLocalisation.dtsx | Chargement référentiel INSEE | SCD2 |
| DimConditionsAccident.dtsx | Chargement conditions météo/surface | SCD1 |
| DimTopographie.dtsx | Chargement profil topographique | SCD1 |
| DimTypeVehicule.dtsx | Chargement types de véhicules | SCD1 |
| DimCollision.dtsx | Chargement profils de collision | SCD1 |
| DimConducteur.dtsx | Chargement profils conducteurs | SCD2 |
| DimProfilVictime.dtsx | Chargement profils victimes | SCD2 |
| DimRadar.dtsx | Chargement radars fixes France | SCD2 |
| FactAccident.dtsx | Chargement faits accidents | Fait |
| FactVehicule.dtsx | Chargement faits véhicules/victimes | Fait |

---

## 🛠️ Technologies
- **SQL Server 2019**
- **SSIS** — SQL Server Integration Services (Visual Studio 2019)
- **T-SQL** — Création dimensions temporelles, vues sources, monitoring ETL
- **Star Schema** — Modélisation dimensionnelle
