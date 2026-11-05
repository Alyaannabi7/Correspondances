# Correspondances


Méthode Merise : MCD → MLD → MPD (PostgreSQL 16).
Images : `docs/mcd.png`, `docs/mld.png`, `docs/mpd.png`.
Sources : `mcd.mcd` (Mocodo, génère MCD + MLD), `mpd.mmd` (Mermaid), script SQL V1.

> Les noms du MCD sont en anglais (STOP, BOOKING…) car ils deviennent directement les tables et entités JPA.

---

## 1. MCD (Modèle Conceptuel de Données)

### Entités

| Entité | Identifiant | Propriétés |
|---|---|---|
| STATION (gare) | station_id | code, name, city |
| LINE (ligne) | line_id | code, name |
| SERVICE_CALENDAR (régime) | calendar_id | code, label, monday…sunday, valid_from, valid_to |
| EXCEPTION_DAY (jour) | exception_date | — |
| TRAIN | train_id | number |
| STOP (desserte, **entité faible**) | train_id + stop_order | arrival_time, departure_time |
| CAR (voiture) | car_id | car_number |
| SEAT (place) | seat_id | seat_number |
| COMFORT_CLASS (classe) | class_id | code, label |
| FARE_PERIOD (période tarifaire) | period_id | code, label |
| PASSENGER_PROFILE (profil) | profile_id | code, label |
| FARE (tarif) | fare_id | price_per_km |
| PASSENGER (voyageur) | passenger_id | first_name, last_name, email, birth_date |
| BOOKING (réservation) | booking_id | travel_date, price, status, booked_at |

### Associations et cardinalités

| Association | Entre | Cardinalités | Données portées |
|---|---|---|---|
| **BELONGS_TO** | LINE — STATION | LINE 2,n / STATION 1,n | position, km_from_origin |
| RUNS_ON | TRAIN — LINE | TRAIN 1,1 / LINE 0,n | — |
| FOLLOWS | TRAIN — SERVICE_CALENDAR | TRAIN 1,1 / CALENDAR 0,n | — |
| **HAS_EXCEPTION** | SERVICE_CALENDAR — EXCEPTION_DAY | 0,n / 0,n | exception_type (ADDED / REMOVED) |
| **SERVES** (identifiante) | STOP — TRAIN | STOP (1,1) / TRAIN 2,n | — |
| AT_STATION | STOP — STATION | STOP 1,1 / STATION 0,n | — |
| COMPOSED_OF | CAR — TRAIN | CAR 1,1 / TRAIN 1,n | — |
| CONTAINS | SEAT — CAR | SEAT 1,1 / CAR 1,n | — |
| HAS_CLASS | SEAT — COMFORT_CLASS | SEAT 1,1 / CLASS 0,n | — |
| FOR_CLASS / FOR_PERIOD / FOR_PROFILE | FARE — chaque dimension | FARE 1,1 / dimension 0,n | — |
| MAKES | BOOKING — PASSENGER | BOOKING 1,1 / PASSENGER 0,n | — |
| RESERVES | BOOKING — SEAT | BOOKING 1,1 / SEAT 0,n | — |
| **DEPARTS_FROM** | BOOKING — STOP | BOOKING 1,1 / STOP 0,n | — |
| **ARRIVES_AT** | BOOKING — STOP | BOOKING 1,1 / STOP 0,n | — |
| APPLIES | BOOKING — FARE | BOOKING 1,1 / FARE 0,n | — |

### Choix expliqués

- **BELONGS_TO porte des données** : l'ordre d'une gare et son kilométrage n'appartiennent ni à la gare (elle est sur plusieurs lignes) ni à la ligne. C'est le kilométrage qui donne la **distance** d'un trajet (km arrivée − km départ), donc le prix.
- **La desserte (STOP) porte des données** : les horaires n'existent que pour le couple (train, gare), ce n'est pas une simple FK. Au départ c'est une association n,n TRAIN — STATION ; elle est **réifiée en entité faible** (identifiée par son train + son ordre de passage) parce que la réservation doit pouvoir pointer dessus (gare de montée, gare de descente). Un train direct n'a simplement pas de STOP pour les gares qu'il saute.
- **Le tarif dépend de trois dimensions à la fois** (classe × période × profil) : c'est une ternaire, réifiée en entité FARE (reliée 1,1 à chaque dimension) pour que la réservation puisse la référencer. L'unicité du triplet est garantie au niveau physique par un `UNIQUE`. La 4e dimension, la distance, n'est pas stockée : elle se calcule.
- **Calendrier** : un régime (jours de semaine cochés) + des exceptions datées. Ça couvre « tous les jours », « semaine », « week-end » et « jours exceptionnels » sans dupliquer les trains.
- **La réservation porte sur un SEGMENT** : elle pointe vers une desserte de départ et une desserte d'arrivée, pas vers « le train ». C'est ce qui permet de revendre la place après la descente du voyageur.
- **Le prix payé est copié dans la réservation** : un tarif peut changer, le prix d'un billet déjà vendu, non.

---

## 2. MLD (Modèle Logique de Données)

Clé primaire **soulignée** (ici en gras), clé étrangère précédée de `#`.

```
station        (**id**, code, name, city)
line           (**id**, code, name)
line_station   (**#line_id, #station_id**, position, km_from_origin)
service_calendar (**id**, code, label, monday, …, sunday, valid_from, valid_to)
calendar_exception (**#calendar_id, exception_date**, exception_type)
train          (**id**, number, #line_id, #calendar_id)
stop           (**#train_id, stop_order**, #line_id, #station_id, arrival_time, departure_time)
comfort_class  (**id**, code, label)
car            (**id**, #train_id, car_number)
seat           (**id**, #car_id, seat_number, #comfort_class_id)
fare_period    (**id**, code, label)
passenger_profile (**id**, code, label)
fare           (**id**, #comfort_class_id, #fare_period_id, #passenger_profile_id, price_per_km)
passenger      (**id**, first_name, last_name, email, birth_date, created_at)
booking        (**id**, #passenger_id, #seat_id, #train_id, travel_date,
                #from_stop_order, #to_stop_order, #fare_id, price, status, booked_at)
```

### Règles de passage appliquées

- Association **1,1 — 0,n** → la clé de l'entité côté « n » migre comme FK côté « 1,1 » (ex : `train.line_id`).
- Association **n,n** avec données → devient une table dont la PK est la combinaison des FK (`line_station`, `stop`).
- Ternaire TARIFER → table `fare` ; une PK technique `id` est ajoutée (plus simple pour la FK depuis `booking`), et l'unicité des trois dimensions est garantie par une contrainte `UNIQUE`.
- `stop` est identifiée par `(train_id, stop_order)` : c'est cette clé que `booking` référence pour le départ et l'arrivée.

---

## 3. MPD (Modèle Physique — PostgreSQL 16)

Script : `src/main/resources/db/migration/V1__init_schema.sql` (exécuté par Flyway).

Écarts MLD → MPD (choix physiques) :
- `BELONGS_TO` → table `line_station`, `HAS_EXCEPTION` → `calendar_exception`.
- `stop` reçoit aussi `line_id` : sert à une FK composite qui garantit que la gare desservie est bien sur la ligne du train.
- Dans `booking`, `train_id 1` et `train_id 2` du MLD sont fusionnés en un seul `train_id` : départ et arrivée sont forcément dans le même train.

### Types

- PK `BIGINT GENERATED ALWAYS AS IDENTITY` (standard SQL, recommandé sur PG ≥ 10 plutôt que `SERIAL`). `SMALLINT` pour les petites tables de référence.
- Montants en `NUMERIC` (jamais `FLOAT` pour de l'argent : erreurs d'arrondi).
- Horaires en `TIME`, dates de voyage en `DATE`, horodatages en `TIMESTAMPTZ`.

### Contraintes et justification

| Règle métier | Contrainte |
|---|---|
| Une place n'est pas réservée deux fois sur le même segment/date | `UNIQUE (seat_id, travel_date, from_stop_order)` |
| Deux segments d'une même place ne se **chevauchent** pas (Marseille→Pertuis bloque Aix→Pertuis) | `EXCLUDE USING gist (seat_id =, travel_date =, int4range(from, to) &&)` sur les réservations confirmées |
| Un segment va vers l'avant | `CHECK (from_stop_order < to_stop_order)` |
| La gare desservie appartient à la ligne du train | FK composite `stop(line_id, station_id) → line_station` |
| Le train de la desserte est bien sur cette ligne | FK composite `stop(train_id, line_id) → train(id, line_id)` |
| Arrivée ≤ départ dans une même gare | `CHECK` |
| Arrivée en gare N+1 après le départ de la gare N | trigger de contrainte `DEFERRABLE` (une `CHECK` ne voit qu'une ligne) |
| Prix ≥ 0, n° de voiture / place > 0 | `CHECK` |
| Une combinaison classe/période/profil = un seul tarif | `UNIQUE` composite |

**Pourquoi l'UNIQUE ne suffit pas** : il empêche deux réservations qui commencent au même arrêt, mais pas `[1→3]` + `[2→3]`. L'intervalle `[from, to)` est semi-ouvert : la place réservée `1→2` est libre pour `2→3`, c'est exactement « le voyageur descend, la place se revend ».

### Comportements à la suppression

- `CASCADE` quand l'enfant n'a aucun sens sans le parent : tracé d'une ligne, dessertes et voitures d'un train, places d'une voiture, exceptions d'un calendrier.
- `RESTRICT` dès qu'une réservation (donc une vente) est en jeu, et pour les gares/tarifs/référentiels : on ne perd jamais l'historique commercial par effet de bord.

### Limites connues (assumées)

- Les trains de nuit qui passent minuit ne sont pas gérés par le type `TIME` (hors périmètre : réseau régional).
- « La réservation pointe vers une circulation qui existe ce jour-là » dépend du calendrier : c'est garanti par le générateur, et pourrait l'être par un trigger.
