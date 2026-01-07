# Analýza U.S. ZIP Code Metadata pomocou ELT v Snowflake
## 1️⃣ Úvod a popis zdrojových dát
V tomto projekte analyzujeme **geografické a demografické údaje o ZIP kódoch v Spojených štátoch amerických.** Cieľom analýzy je porozumieť:
- rozloženiu populácie v jednotlivých štátoch a regiónoch,
- demografickým charakteristikám obyvateľstva (vek, pohlavie),
- významu mediálnych oblastí (DMA),
- geografickému rozloženiu ZIP kódov.
Zdrojové dáta pochádzajú zo Snowflake Marketplace, konkrétne z datasetu U.S. ZIP Code Metadata, ktorý poskytuje agregované demografické a geografické informácie na úrovni ZIP kódov.
Dataset obsahuje jednu hlavnú tabuľku:
**ZIP_CODE_METADATA** – obsahuje informácie o ZIP kódoch, mestách, štátoch, geografickej polohe (latitude, longitude), časových pásmach a demografických ukazovateľoch (celková populácia, medián veku, populácia mužov a žien).
Účelom ELT procesu bolo tieto dáta očistiť, transformovať a reorganizovať do dimenzionálneho modelu typu Star Schema, vhodného pre analytické dotazy a vizualizácie.

---
## 1️⃣.1 Dátová architektúra – pôvodná schéma
Zdrojový dataset má jednoduchú relačnú štruktúru pozostávajúcu z jednej tabuľky. Entitno-relačný diagram (ERD) pôvodnej schémy je znázornený na obrázku nižšie
**ERD Schema – zdrojové dáta**
![ERD SCHEMA](img/erd_schema.png)

---
## 2️⃣ Dimenzionálny model
Na základe zdrojových dát bol navrhnutý dimenzionálny model typu Star Schema podľa Kimballovej metodológie. Model pozostáva z 1 faktovej tabuľky a 5 dimenzií.

**Faktová tabuľka**
- FACT_ZIP_USA_SNAPSHOT
Obsahuje agregované metriky o ZIP kódoch a prepojenia na všetky dimenzie.
Hlavné metriky:
- počet ZIP kódov,
- poradie ZIP kódu podľa populácie v rámci štátu (window function),
- celková populácia štátu,
- populácia predchádzajúceho ZIP kódu (LAG).

**Dimenzie**
- DIM_ZIP_USA – informácie o ZIP kódoch, mestách a štátoch (SCD Typ 0),
- DIM_GEO_USA – geografické údaje (latitude, longitude, časové pásmo, daylight savings) (SCD Typ 0),
- DIM_DMA_USA – mediálne oblasti DMA (SCD Typ 0),
- DIM_DEMOGRAPHICS_USA – demografické údaje o populácii (SCD Typ 0),
- DIM_LOAD_DATE – dátum načítania dát (SCD Typ 0).
Štruktúra hviezdicového modelu je znázornená na diagrame nižšie.
**Star Schema**
![Star SCHEMA](img/star_schema.png)

---
## 3️⃣ ELT proces v Snowflake
ELT proces bol implementovaný v prostredí **Snowflake** s cieľom transformovať zdrojové dáta zo Snowflake Marketplace do dimenzionálneho modelu typu Star Schema, vhodného na analytické dotazy a vizualizácie. Proces pozostáva z troch hlavných fáz: **Extract, Load a Transform.**

### 3️⃣.1 Extract (Extrahovanie dát)
Zdrojové dáta pochádzajú výhradne zo Snowflake Marketplace, konkrétne z datasetu **U.S. ZIP Code Metadata**. Tento dataset poskytuje agregované geografické a demografické údaje na úrovni ZIP kódov v USA.
Keďže Marketplace dataset je už dostupný priamo v Snowflake, nebolo potrebné nahrávať externé súbory (.csv) ani vytvárať samostatné staging tabuľky pre každú entitu, ako je to bežné pri práci s viacerými zdrojovými tabuľkami. Zdrojová tabuľka bola skopírovaná priamo do staging vrstvy jedným SQL príkazom.
Príklad kódu:
```sql
CREATE OR REPLACE TABLE ZIP_CODE_METADATA_STAGING AS
SELECT *
FROM U_S__ZIP_CODE_METADATA.ZIP_DEMOGRAPHICS.ZIP_CODE_METADATA;
```
Tento prístup zjednodušil proces extrakcie dát a zabezpečil, že staging vrstva obsahuje presnú kópiu zdrojových dát bez potreby manuálneho importu.

### 3️⃣.2 Load (Načítanie dát)
V rámci fázy Load boli dáta zo staging tabuľky presunuté do ďalších vrstiev spracovania. Pred samotnou transformáciou prebehla základná validácia dát, ktorá zahŕňala:
- kontrolu počtu záznamov,
- overenie unikátnosti ZIP kódov,
- identifikáciu NULL hodnôt v kľúčových atribútoch,
- základnú deduplikáciu.
Následne bola vytvorená očistená verzia dát `(ZIP_CODE_METADATA_CLEAN)`, ktorá slúžila ako vstup pre tvorbu dimenzií a faktovej tabuľky.
Tento krok zabezpečil, že do dimenzionálneho modelu vstupujú konzistentné a kvalitné dáta, čo je kľúčové pre správne analytické výsledky.
```sql
-- Validácie
SELECT COUNT(*) AS rows_clean FROM ZIP_CODE_METADATA_CLEAN;

SELECT COUNT(*) AS distinct_zip
FROM (SELECT DISTINCT ZIP FROM ZIP_CODE_METADATA_CLEAN);

SELECT ZIP, COUNT(*) cnt
FROM ZIP_CODE_METADATA_CLEAN
GROUP BY ZIP
HAVING COUNT(*) > 1;
```

### 3️⃣.3 Transform (Transformácia dát)
Transformačná fáza predstavuje najdôležitejšiu časť ELT procesu. V tejto fáze boli dáta zo staging vrstvy **vyčistené, transformované a obohatené** s cieľom vytvoriť dimenzie a faktovú tabuľku.
#### Transformácia a tvorba dimenzií
Dimenzie boli navrhnuté tak, aby poskytovali kontext pre faktovú tabuľku a umožnili analytické pohľady z rôznych perspektív.

#### DIM_ZIP_USA
Táto dimenzia obsahuje základné informácie o ZIP kódoch, mestách a štátoch. Každý ZIP kód je reprezentovaný jedinečným surrogate kľúčom. Keďže údaje o ZIP kódoch sú považované za statické, dimenzia je klasifikovaná ako **SCD Typ 0**.
```sql
CREATE OR REPLACE TABLE DIM_ZIP_USA AS
SELECT
  ROW_NUMBER() OVER (ORDER BY ZIP) AS ZIP_USA_ID,
  ZIP,
  CITY,
  STATE
FROM (
  SELECT DISTINCT ZIP, CITY, STATE
  FROM ROOSTER_PROJECT_DB.STAGING.ZIP_CODE_METADATA_CLEAN
);
```

#### DIM_GEO_USA
Dimenzia uchováva geografické údaje, ako sú zemepisná šírka, dĺžka, časové pásmo a informácia o daylight savings. Tieto údaje umožňujú geografické a mapové analýzy. Dimenzia je navrhnutá ako **SCD Typ 0**, pretože geografické súradnice sa v čase nemenia.
```sql
CREATE OR REPLACE TABLE DIM_GEO_USA AS
SELECT
  ROW_NUMBER() OVER (
    ORDER BY LATITUDE, LONGITUDE, TIMEZONE, IS_DAYLIGHT_SAVINGS
  ) AS GEO_USA_ID,
  LATITUDE,
  LONGITUDE,
  TIMEZONE,
  IS_DAYLIGHT_SAVINGS
FROM (
  SELECT DISTINCT
    LATITUDE, LONGITUDE, TIMEZONE, IS_DAYLIGHT_SAVINGS
  FROM ROOSTER_PROJECT_DB.STAGING.ZIP_CODE_METADATA_CLEAN
);
```

#### DIM_DMA_USA
Táto dimenzia reprezentuje mediálne oblasti (Designated Market Areas – DMA). Umožňuje analyzovať populáciu a ZIP kódy z pohľadu mediálnych regiónov. Dimenzia je typu **SCD Typ 0**.
```sql
CREATE OR REPLACE TABLE DIM_DMA_USA AS
SELECT
  ROW_NUMBER() OVER (ORDER BY DMA_ID, DMA_NAME) AS DMA_USA_ID,
  DMA_ID,
  DMA_NAME
FROM (
  SELECT DISTINCT DMA_ID, DMA_NAME
  FROM ROOSTER_PROJECT_DB.STAGING.ZIP_CODE_METADATA_CLEAN
);
```

#### DIM_DEMOGRAPHICS_USA
Dimenzia obsahuje demografické údaje o populácii, ako je celková populácia, medián veku a rozdelenie populácie podľa pohlavia. Tieto údaje poskytujú dôležitý kontext pre analýzu obyvateľstva jednotlivých oblastí. Dimenzia je klasifikovaná ako **SCD Typ 0**, keďže pracuje s agregovanými, nemennými hodnotami.
```sql
CREATE OR REPLACE TABLE DIM_DEMOGRAPHICS_USA AS
SELECT
  ROW_NUMBER() OVER (
    ORDER BY TOTAL_POPULATION, MEDIAN_AGE
  ) AS DEMOGRAPHICS_USA_ID,
  TOTAL_POPULATION,
  MEDIAN_AGE,
  TOTAL_MALE_POPULATION,
  TOTAL_FEMALE_POPULATION
FROM (
  SELECT DISTINCT
    TOTAL_POPULATION,
    MEDIAN_AGE,
    TOTAL_MALE_POPULATION,
    TOTAL_FEMALE_POPULATION
  FROM ROOSTER_PROJECT_DB.STAGING.ZIP_CODE_METADATA_CLEAN
);
```

#### DIM_LOAD_DATE
Dimenzia slúži na evidenciu dátumu načítania dát do dátového skladu. Umožňuje sledovať snapshoty dát v čase. Ide o **SCD Typ 0**, kde sa nové záznamy pridávajú pri každom novom načítaní.
```sql
CREATE OR REPLACE TABLE DIM_LOAD_DATE AS
SELECT
  1 AS LOAD_DATE_ID,
  CURRENT_DATE() AS LOAD_DATE;
```

### Faktová tabuľka `FACT_ZIP_USA_SNAPSHOT` – tvorba a funkcia
Faktová tabuľka `FACT_ZIP_USA_SNAPSHOT` predstavuje **centrálny bod hviezdicovej schémy (Star Schema)**. Jej úlohou je:
- ukladať merateľné metriky (measures) súvisiace so ZIP kódmi,
- obsahovať cudzie kľúče do dimenzií (ZIP, geografia, DMA, demografia, dátum načítania),
- umožniť rýchle analytické dotazy typu „top ZIP“, „porovnanie štátov“, „populácia podľa DMA“, „geografická vizualizácia“.
Keďže zdrojový dataset neobsahuje transakcie v čase (napr. udalosti), faktová tabuľka je navrhnutá ako **snapshot** – t. j. opisuje stav dát v momente načítania (načítanie je reprezentované dimenziou `DIM_LOAD_DATE`). V prípade budúcich refreshov datasetu by tak bolo možné porovnávať snapshoty medzi rôznymi dňami načítania.
**Príklad kódu:**
```sql
CREATE OR REPLACE TABLE FACT_ZIP_USA_SNAPSHOT AS
WITH base AS (
  SELECT
    c.ZIP,
    c.STATE,
    c.DMA_ID,
    c.DMA_NAME,
    c.LATITUDE,
    c.LONGITUDE,
    c.TIMEZONE,
    c.IS_DAYLIGHT_SAVINGS,
    c.TOTAL_POPULATION,
    c.MEDIAN_AGE,
    c.TOTAL_MALE_POPULATION,
    c.TOTAL_FEMALE_POPULATION,
    1 AS ZIP_COUNT,
    RANK() OVER (
      PARTITION BY c.STATE
      ORDER BY c.TOTAL_POPULATION DESC NULLS LAST
    ) AS POPULATION_RANK_IN_STATE,
    SUM(c.TOTAL_POPULATION) OVER (
      PARTITION BY c.STATE
    ) AS STATE_TOTAL_POPULATION,
    LAG(c.TOTAL_POPULATION) OVER (
      PARTITION BY c.STATE
      ORDER BY c.ZIP
    ) AS PREVIOUS_ZIP_POPULATION
  FROM ROOSTER_PROJECT_DB.STAGING.ZIP_CODE_METADATA_CLEAN c
)
SELECT
  ROW_NUMBER() OVER (ORDER BY b.ZIP) AS FACT_ZIP_USA_SNAPSHOT_ID,
  z.ZIP_USA_ID,
  g.GEO_USA_ID,
  d.DMA_USA_ID,
  demo.DEMOGRAPHICS_USA_ID,
  ld.LOAD_DATE_ID,
  -- metriky
  b.ZIP_COUNT,
  b.POPULATION_RANK_IN_STATE,
  b.STATE_TOTAL_POPULATION,
  b.PREVIOUS_ZIP_POPULATION
FROM base b
JOIN DIM_ZIP_USA z ON b.ZIP = z.ZIP
JOIN DIM_GEO_USA g ON b.LATITUDE = g.LATITUDE
 AND b.LONGITUDE = g.LONGITUDE
 AND NVL(b.TIMEZONE,'') = NVL(g.TIMEZONE,'')
 AND b.IS_DAYLIGHT_SAVINGS = g.IS_DAYLIGHT_SAVINGS
JOIN DIM_DMA_USA d ON NVL(b.DMA_ID,-1) = NVL(d.DMA_ID,-1)
 AND NVL(b.DMA_NAME,'') = NVL(d.DMA_NAME,'')
JOIN DIM_DEMOGRAPHICS_USA demo ON NVL(b.TOTAL_POPULATION,-1) = NVL(demo.TOTAL_POPULATION,-1)
 AND NVL(b.MEDIAN_AGE,-1) = NVL(demo.MEDIAN_AGE,-1)
 AND NVL(b.TOTAL_MALE_POPULATION,-1) = NVL(demo.TOTAL_MALE_POPULATION,-1)
 AND NVL(b.TOTAL_FEMALE_POPULATION,-1) = NVL(demo.TOTAL_FEMALE_POPULATION,-1)
JOIN DIM_LOAD_DATE ld ON 1=1;
```

#### 1) Prečo sa používa CTE `base`
Vytvorenie faktovej tabuľky prebieha v dvoch logických krokoch. Prvý krok je definovaný pomocou CTE:
```sql
WITH base AS ( ... )
```
CTE `base` slúži ako prípravná vrstva, kde sa:
- vyberajú relevantné atribúty zo staging tabuľky `ZIP_CODE_METADATA_CLEAN`,
- počítajú metriky a analytické ukazovatele,
- aplikujú povinné window functions.
Výhodou je, že celý výpočet metrík je oddelený od samotného mapovania na dimenzie, čo zvyšuje čitateľnosť a kontrolu správnosti.

#### 2) Výber atribútov zo staging vrstvy
V `base` sa najprv preberú identifikačné a popisné údaje zo staging tabuľky:
- `ZIP`, `STATE` – identifikácia ZIP kódu a štátu,
- `DMA_ID`, `DMA_NAME` – mediálny región (DMA),
- `LATITUDE`, `LONGITUDE`, `TIMEZONE`, `IS_DAYLIGHT_SAVINGS` – geografický kontext,
- `TOTAL_POPULATION`, `MEDIAN_AGE`, `TOTAL_MALE_POPULATION`, `TOTAL_FEMALE_POPULATION` – demografické hodnoty.
Tieto polia sa následne používajú na:
- výpočet metrík,
- priradenie surrogate kľúčov z dimenzií.

#### 3) Metriky vo faktovej tabuľke
Faktová tabuľka obsahuje tieto metriky (measures):

`ZIP_COUNT`
```sql
1 AS ZIP_COUNT
```
Táto metrika má v každom riadku hodnotu 1. V analytike je veľmi užitočná, pretože umožňuje rýchlo počítať:
-  počet ZIP kódov v štáte (`SUM(ZIP_COUNT)`),
- počet ZIP kódov v DMA (`SUM(ZIP_COUNT)`),
- počet ZIP kódov v časovom pásme a pod.
Je to typická technika pre „count measure“.

#### 4) Povinné window functions – význam a dôvod použitia
Zadanie vyžaduje použitie window functions vo faktovej tabuľke. V tomto projekte sú použité tri:
#### A) `RANK()` – poradie ZIP podľa populácie v rámci štátu
```sql
RANK() OVER (
  PARTITION BY c.STATE
  ORDER BY c.TOTAL_POPULATION DESC NULLS LAST
) AS POPULATION_RANK_IN_STATE
```
- `PARTITION BY STATE` znamená, že poradie sa počíta samostatne pre každý štát.
- `ORDER BY TOTAL_POPULATION DESC` zoradí ZIPy od najľudnatejšieho.
- výsledok umožňuje vizualizácie typu:
  - „Top ZIP v každom štáte“
  - „Najväčšie ZIPy podľa populácie“.

#### B) `SUM() OVER` – celková populácia štátu
```sql
SUM(c.TOTAL_POPULATION) OVER (
  PARTITION BY c.STATE
) AS STATE_TOTAL_POPULATION
```
Tento stĺpec opakuje v každom ZIP riadku hodnotu **celkovej populácie celého štátu** (vypočítanú zo ZIP záznamov). Je užitočný napríklad pre:
- výpočet podielu ZIP populácie na štáte,
- porovnávanie ZIP v rámci štátu bez ďalších joinov/agregácií.

#### C) `LAG()` – populácia predchádzajúceho ZIP v rámci štátu
```sql
LAG(c.TOTAL_POPULATION) OVER (
  PARTITION BY c.STATE
  ORDER BY c.ZIP
) AS PREVIOUS_ZIP_POPULATION
```
`LAG` vráti hodnotu z predchádzajúceho riadku v rámci rovnakého štátu, zoradeného podľa ZIP. Umožňuje analyzovať:
- lokálne rozdiely medzi susednými ZIPmi v poradí,
- jednoduché trendové porovnanie v rámci štátu.
Aj keď ZIP kódy nie sú geograficky zoradené, ide o validnú ukážku analytickej funkcie nad dátami a spĺňa požiadavku projektu.

#### 5) Druhá časť – mapovanie na dimenzie (surrogate keys)
Po príprave dát v `base` nasleduje druhý logický krok:
prepojenie na dimenzie a nahradenie prirodzených kľúčov surrogate kľúčmi.
Primárny kľúč faktu
```sql
ROW_NUMBER() OVER (ORDER BY b.ZIP) AS FACT_ZIP_USA_SNAPSHOT_ID
```
`ROW_NUMBER()` vygeneruje jedinečné ID každého riadku vo faktoch. Je to technika vhodná pre „snapshot“ fakt.

#### 6) JOINy na dimenzie – prečo sú tam NVL
Fakt sa pripája na dimenzie tak, aby sa do faktovej tabuľky uložili iba surrogate kľúče:
- `DIM_ZIP_USA` podľa `ZIP`
- `DIM_GEO_USA` podľa `LATITUDE`, `LONGITUDE`, `TIMEZONE`, `IS_DAYLIGHT_SAVINGS`
- `DIM_DMA_USA` podľa `DMA_ID`, `DMA_NAME`
- `DIM_DEMOGRAPHICS_USA` podľa demografických metrík
- `DIM_LOAD_DATE` cez `ON 1=1` (ide o konštantný snapshot dátumu načítania)
Použitie `NVL()` je dôležité, pretože v dátach sa môžu vyskytnúť NULL hodnoty (napr. DMA nemusí byť uvedené pre všetky ZIP). `NVL` zabezpečí, že porovnanie je konzistentné a join nevypadne kvôli NULL porovnaniu.
Príklad:
```sql
ON NVL(b.DMA_ID,-1) = NVL(d.DMA_ID,-1)
```

#### 7) Čo umožňuje výsledná faktová tabuľka
Výsledná tabuľka `FACT_ZIP_USA_SNAPSHOT` umožňuje:
- analýzy populácie podľa štátu, mesta a DMA,
- výpočet „top“ ZIP kódov podľa populácie,
- geografické vizualizácie (map/scatter) cez lat/long dimenziu,
- analýzu mediánu veku a pohlavného rozdelenia naprieč USA,
- rýchle agregácie cez `ZIP_COUNT`.

---
### 4️⃣ Vizualizácia dát
Vizualizácia dát slúži ako posledná vrstva analytického procesu, ktorej cieľom je **prehľadne prezentovať výsledky viacdimenzionálneho modelu** vytvoreného v Snowflake. Všetky grafy vychádzajú z faktovej tabuľky `FACT_ZIP_USA_SNAPSHOT`, ktorá je prepojená s dimenziami pomocou surrogate kľúčov, čo zabezpečuje konzistentné a výkonné analytické dotazy.
![DASHBOARD](img/project_dashboard.png)

#### Graf 1: Počet ZIP kódov podľa štátu
Táto vizualizácia zobrazuje **počet ZIP kódov v jednotlivých štátoch USA**. Cieľom grafu je porovnať administratívne členenie štátov a identifikovať, ktoré štáty majú najväčší počet ZIP oblastí.
Graf využíva dimenziu `DIM_ZIP_USA` a agregáciu nad faktovou tabuľkou. Výsledky ukazujú výrazné rozdiely medzi jednotlivými štátmi, čo môže súvisieť s rozlohou štátu, hustotou obyvateľstva alebo urbanizáciou.
Táto vizualizácia poskytuje základný kontext pre ďalšie analýzy populácie a demografie.
```sql
SELECT z.STATE, COUNT(*) AS ZIP_COUNT
FROM ROOSTER_PROJECT_DB.DW.FACT_ZIP_USA_SNAPSHOT f
JOIN ROOSTER_PROJECT_DB.DW.DIM_ZIP_USA z
  ON f.ZIP_USA_ID = z.ZIP_USA_ID
GROUP BY z.STATE
ORDER BY ZIP_COUNT DESC;
```

#### Graf 2: Top 20 ZIP kódov podľa celkovej populácie
Druhý graf zobrazuje **20 ZIP kódov s najvyšším počtom obyvateľov**. Vizualizácia umožňuje identifikovať najľudnatejšie oblasti v rámci USA a porovnať ich veľkosť populácie.
Použitá metrika `TOTAL_POPULATION` pochádza z dimenzie `DIM_DEMOGRAPHICS_USA`, pričom poradie ZIP kódov je určené pomocou analytickej funkcie `RANK()` vypočítanej vo faktovej tabuľke.
Graf je vhodný najmä na:
- identifikáciu demograficky významných oblastí,
- plánovanie marketingových alebo logistických aktivít,
- regionálne porovnávanie hustoty obyvateľstva.
```sql
SELECT
  z.ZIP,
  z.CITY,
  z.STATE,
  d.TOTAL_POPULATION
FROM ROOSTER_PROJECT_DB.DW.FACT_ZIP_USA_SNAPSHOT f
JOIN ROOSTER_PROJECT_DB.DW.DIM_ZIP_USA z
  ON f.ZIP_USA_ID = z.ZIP_USA_ID
JOIN ROOSTER_PROJECT_DB.DW.DIM_DEMOGRAPHICS_USA d
  ON f.DEMOGRAPHICS_USA_ID = d.DEMOGRAPHICS_USA_ID
ORDER BY d.TOTAL_POPULATION DESC NULLS LAST
LIMIT 20;
```

#### Graf 3: Top 20 DMA oblastí podľa celkovej populácie
Táto vizualizácia agreguje populáciu na úroveň **DMA (Designated Market Area)**, čo predstavuje mediálne regióny používané najmä v reklame a marketingu.
Graf sumarizuje hodnoty `TOTAL_POPULATION` pre jednotlivé DMA oblasti a zobrazuje 20 najväčších regiónov. Vďaka tomu je možné:
- porovnať veľkosť mediálnych trhov,
- identifikovať oblasti s najväčším dosahom,
- analyzovať regionálne rozdiely na vyššej úrovni agregácie než ZIP kód.
Vizualizácia demonštruje silu dimenzionálneho modelu, ktorý umožňuje jednoduché agregácie naprieč rôznymi dimenziami bez potreby zložitých **JOIN** operácií.
```sql
SELECT
  dma.DMA_NAME,
  SUM(d.TOTAL_POPULATION) AS TOTAL_POPULATION
FROM ROOSTER_PROJECT_DB.DW.FACT_ZIP_USA_SNAPSHOT f
JOIN ROOSTER_PROJECT_DB.DW.DIM_DMA_USA dma
  ON f.DMA_USA_ID = dma.DMA_USA_ID
JOIN ROOSTER_PROJECT_DB.DW.DIM_DEMOGRAPHICS_USA d
  ON f.DEMOGRAPHICS_USA_ID = d.DEMOGRAPHICS_USA_ID
GROUP BY dma.DMA_NAME
ORDER BY TOTAL_POPULATION DESC NULLS LAST
LIMIT 20;
```

#### Graf 4: Geografická mapa ZIP kódov podľa štátu (Latitude / Longitude)
Štvrtá vizualizácia je **mapová reprezentácia ZIP kódov** založená na zemepisnej šírke a dĺžke z dimenzie `DIM_GEO_USA`. Každý bod na mape reprezentuje jeden ZIP kód a je priradený ku konkrétnemu štátu.
Cieľom mapy je:
- vizuálne znázorniť geografické rozloženie ZIP kódov,
- overiť konzistenciu geografických dát,
- poskytnúť priestorový kontext pre demografické analýzy.
```sql
SELECT
  dma.DMA_NAME AS DMA,
  SUM(d.TOTAL_POPULATION) AS POPULATION
FROM ROOSTER_PROJECT_DB.DW.FACT_ZIP_USA_SNAPSHOT f
JOIN ROOSTER_PROJECT_DB.DW.DIM_DMA_USA dma
  ON f.DMA_USA_ID = dma.DMA_USA_ID
JOIN ROOSTER_PROJECT_DB.DW.DIM_DEMOGRAPHICS_USA d
  ON f.DEMOGRAPHICS_USA_ID = d.DEMOGRAPHICS_USA_ID
GROUP BY dma.DMA_NAME
ORDER BY POPULATION DESC
LIMIT 15;
```

#### Graf 5: Rozdelenie ZIP kódov podľa mediánu veku (Histogram)
Posledný graf je histogram, ktorý zobrazuje rozdelenie **ZIP kódov podľa mediánu veku obyvateľstva**. Medián veku je zaokrúhlený do intervalov (bucketov), čím vznikajú vekové skupiny.
Vizualizácia umožňuje:
- analyzovať vekovú štruktúru populácie,
- identifikovať oblasti s mladšou alebo staršou populáciou,
- porovnať demografické trendy medzi regiónmi.
```sql
SELECT
  FLOOR(d.MEDIAN_AGE) AS AGE_BUCKET,
  COUNT(*) AS ZIP_COUNT
FROM ROOSTER_PROJECT_DB.DW.FACT_ZIP_USA_SNAPSHOT f
JOIN ROOSTER_PROJECT_DB.DW.DIM_DEMOGRAPHICS_USA d
  ON f.DEMOGRAPHICS_USA_ID = d.DEMOGRAPHICS_USA_ID
WHERE d.MEDIAN_AGE IS NOT NULL
GROUP BY AGE_BUCKET
ORDER BY AGE_BUCKET;
```

### Zhrnutie vizualizácií
Navrhnuté vizualizácie poskytujú **komplexný pohľad na demografické a geografické charakteristiky USA** na rôznych úrovniach detailu – od jednotlivých ZIP kódov až po mediálne regióny (DMA).

---
Autor: Šimon Šedivý
