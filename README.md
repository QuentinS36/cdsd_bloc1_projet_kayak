# Kayak : où partir en France ?

Certification CDSD RNCP35288 - Bloc 1 : Construction et alimentation d'une infrastructure de gestion de donnée

## Contexte

Kayak veut recommander des destinations à partir de **données réelles de météo et d'hôtels** pour les **35 meilleures villes de France**. Ce projet construit l'infrastructure qui collecte, stocke et nettoie ces données. 

## Objectifs

Pour chacune des 35 villes de la liste : 
1. Collecter les coordonnées GPS 
2. Collecter les données météo de la semaine à venir
3. Récupérer des données relatives aux hôtels disponibles dans cette ville

Les données brutes et les données consolidées sont stockées dans un data lake (bucket S3) + copie locale.
Les données sont ensuite nettoyée et stockées dans un data warehouse (NeonDB SQL) + copie locale et intérrogées pour produire deux cartes : un top 5 des villes avec les meilleures conditions météo et un top 20 des hôtel s'y trouvant. 

## Données

```
- Nominatim : coordonnées GPS 
- Open-Meteo : météo à 7 jours 
- Booking : hôtels (scraping)                                                             
```

**Chaque sauvegarde est faite en même temps en local et dans le cloud** : le lac écrit dans `data_lake/` et dans S3 ; l'entrepôt charge la base Neon (PostgreSQL) et une copie SQLite locale (`kayak_warehouse.db' dans 'data_warehouse/`).

## Livrables produits par le projet

| Livrable | Où le trouver |
|---|---|
| CSV enrichi (météo + hôtels) dans le Data Lake | `enriched/kayak_enriched/date=AAAA-MM-JJ/kayak_enriched.csv`, dans S3 et dans `data_lake/` |
| Base SQL avec les données nettoyées | Neon (référence) et `kayak_warehouse.db` (copie locale) : tables `cities`, `weather_forecast`, `hotels` et vue `city_weather_summary` |
| Carte du Top 5 des destinations | `outputs/maps/top5_destinations.html` |
| Carte du Top 20 des hôtels | `outputs/maps/top20_hotels.html` |

Le notebook vérifie que les volumes sont identiques dans Neon et en local, et que la requête SQL et le fichier enrichi donnent le même Top 20.

## Limites

- Le score météo repose sur le seul code météo (ni température, ni vent).
- Dans la base et le fichier enrichi, les hôtels portent les coordonnées de leur ville (les coordonnées réelles ne sont pas dans la page de recherche mais dans celle de chaque hôtel); 
  les vraies coordonnées font l'objet d'un second scraping et ne sont lues que pour le Top 20 (la 2ème carte).
- Prix, notes et prévisions correspondent au moment de la collecte (séjour simulé de 1 nuit à partir de J+1).
- Si Booking a changé sa page, les sélecteurs de `parse_hotels` sont à adapter.

## Structure du repo

```
cdsd_bloc1_projet_kayak/
├── notebooks/projet_kayak.ipynb 
├── outputs/
    ├── top20_hotels.html
    └── top5_destinations.html
├── .env.example
├── .gitignore
├── README.md  
└── requirements.txt
```
Créés à l'exécution : `data_lake/` (copie locale du lac), `data_warehouse/` (copie locale de l'entrepôt), `outputs/maps/` (cartes HTML, présentes dans le repo pour exemple), `debug_booking.png` (uniquement en cas d'échec du scraping).


## Prérequis

| Prérequis | Détail |
|---|---|
| Python 3.10+ et Google Chrome | Chrome est piloté par Selenium (fenêtre visible, à ne pas fermer pendant le scraping) |
| Compte AWS | clés d'accès avec droits sur S3 (le bucket est créé automatiquement s'il n'existe pas) |
| NeonDB | base PostgreSQL créée dans une **région européenne** |
| Internet | Nominatim, Open-Meteo, Booking.com, AWS, Neon |

## Installation

```powershell
conda create -n cdsd_b1_projet_kayak python=3.13.14 pip -y
conda activate cdsd_b1_projet_kayak
python -m pip install -r requirements.txt
Copy-Item .env.example .env                      
```

Renseigner ensuite le fichier `.env` :

- **AWS** : `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_BUCKET_NAME` et `AWS_REGION` (`eu-west-3` = Paris).
- **Neon** : dans la console Neon, copier la **chaîne de connexion** du projet dans `NEON_DATABASE_URL`.

Ne jamais versionner le fichier `.env`.

## Exécution

Ouvrir `kayak_projet.ipynb` et lancer **Run All**. **Le notebook exécute toujours les trois collectes** (API Nominatim, API Open-Meteo, scraping Booking) puis tout le reste. L'éxécution complète peut durer **environ 15-20 minutes**, dont la majeure partie pour le scraping.

Chaque exécution remplace les tables de Neon. Les fichiers du lac sont rangés par date : S3 étant versionné, une relance le même jour conserve l'ancienne version du fichier.


## Données personnelles et conditions d'usage

- **RGPD** : aucune donnée d'utilisateur n'est collectée. Les avis clients sont exclus.
- **Nominatim** : 1 requête par seconde, `User-Agent` fourni, mention « © contributeurs OpenStreetMap » (ODbL).
- **Open-Meteo** : API gratuite réservée à un usage non commercial.
- **Booking.com** : les CGU interdisent le scraping sans accord écrit. Usage **pédagogique uniquement** (20 hôtels par ville, pauses aléatoires, arrêt après 3 échecs). 

## Sources

Coordonnées : © contributeurs OpenStreetMap via Nominatim · Météo : Open-Meteo.com · Hôtels : Booking.com.
