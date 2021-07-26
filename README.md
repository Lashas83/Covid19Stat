# COVID19 Statistika
## Imunizacijos lygio įmonėse su atlyginimo vidurkio palyginimas

Importuoti šie duomenų rinkiniai naudojant Import Flat File funkciją į [MS SQL server](https://www.microsoft.com/en-us/sql-server/sql-server-2019) duomenų bazę:
* Atlyginimų vidurkiai iš Sodros teikiamų duomenų mėnesiniai duomenų https://atvira.sodra.lt/imones/rinkiniai/index.html 
* Asmenų imunizacijos duomenys pagal darbovietes (https://open-data-ls-osp-sdg.hub.arcgis.com/datasets/db8480e783ed46538ba51fc9b380c809_0/about) 2021-07-24 dienos

Importuota buvo į tokias lenteles:
```sql
CREATE TABLE [dbo].[monthly-2021](
	[Draudėjo_kodas_code] [int] NOT NULL,
	[Juridinių_asmenų_registro_kodas_jarCode] [int] NULL,
	[Pavadinimas_name] [nvarchar](500) NOT NULL,
	[Savivaldybė_kurioje_registruota_municipality] [nvarchar](50) NOT NULL,
	[Ekonominės_veiklos_rūšies_kodas_ecoActCode] [int] NULL,
	[Ekonominės_veiklos_rūšies_pavadinimas_ecoActName] [nvarchar](200) NULL,
	[Mėnuo_month] [int] NOT NULL,
	[Vidutinis_darbo_užmokestis_avgWage] [varchar](50) NULL,
	[Apdraustųjų_skaičius_numInsured] [int] NOT NULL,
	[Vidutinis_darbo_užmokestis_II_avgWage2] [varchar](50) NULL,
	[Apdraustųjų_skaičius_II_numInsured2] [int] NOT NULL,
	[Valstybinio_socialinio_draudimo_įmoka_tax] [varchar](50) NULL
) ON [PRIMARY]
GO
```

```sql
CREATE TABLE [dbo].[Imunizacija_pagal_darbovietes](
	[ESRI_OID] [nvarchar](50) NOT NULL,
	[workplace_code] [int] NOT NULL,
	[workplace_name] [nvarchar](500) NULL,
	[activity_evrk2] [int] NULL,
	[municipality] [nvarchar](100) NOT NULL,
	[county] [nvarchar](100) NOT NULL,
	[employees] [int] NOT NULL,
	[immunised_percent] [nvarchar](50) NOT NULL,
	[percentile_1] [int] NOT NULL,
	[percentile_2] [int] NULL,
	[as_of_date] [datetime2](7) NOT NULL,
 CONSTRAINT [PK_Imunizacija_pagal_darbovietes] PRIMARY KEY CLUSTERED 
(
	[workplace_code] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
) ON [PRIMARY]
GO
```

Toliau praleistas šis skriptas, kurio pagalba sutvarkyti duomenų tipai:
```sql
SELECT Draudėjo_kodas_code AS Code
    ,CAST (Juridinių_asmenų_registro_kodas_jarCode AS INT) AS jarCode
    ,Pavadinimas_name AS name
    ,Savivaldybė_kurioje_registruota_municipality AS municipality
    ,(CASE WHEN LEN(Ekonominės_veiklos_rūšies_pavadinimas_ecoActName) = 0 THEN NULL ELSE Ekonominės_veiklos_rūšies_pavadinimas_ecoActName END) ecoActName
    ,(CASE WHEN Ekonominės_veiklos_rūšies_kodas_ecoActCode = 0 THEN NULL ELSE Ekonominės_veiklos_rūšies_kodas_ecoActCode END) ecoActCode
    ,Mėnuo_month AS [month]
    ,(CASE WHEN LEN(Vidutinis_darbo_užmokestis_avgWage) = 0 THEN NULL ELSE CAST (Vidutinis_darbo_užmokestis_avgWage AS DECIMAL(28,2)) END) avgWage
    ,Apdraustųjų_skaičius_numInsured AS numInsured
    ,(CASE WHEN LEN(Vidutinis_darbo_užmokestis_II_avgWage2) = 0 THEN NULL ELSE CAST (Vidutinis_darbo_užmokestis_II_avgWage2 AS DECIMAL(28,2)) END) avgWage2
    ,Apdraustųjų_skaičius_II_numInsured2 AS numInsured2
	,(CASE WHEN LEN(Valstybinio_socialinio_draudimo_įmoka_tax) = 0 THEN NULL ELSE CAST (Valstybinio_socialinio_draudimo_įmoka_tax AS DECIMAL(28,2)) END) tax
	INTO [monthly-2021-cleaned]
FROM [dbo].[monthly-2021]
WHERE LEN(Juridinių_asmenų_registro_kodas_jarCode) > 0
```

Ir tada paleista ši užklausa:
```sql
WITH WageStats AS 
(SELECT DISTINCT immunised_percent AS [Imunizacijos procentas]
	, MAX(avgWage) OVER (PARTITION BY immunised_percent) [Max Vidutinis atlyginimas]
	, PERCENTILE_DISC ( 0.99 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P99
	, PERCENTILE_DISC ( 0.95 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P95
	, PERCENTILE_DISC ( 0.75 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P75
	, PERCENTILE_DISC ( 0.5 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P50 
	, PERCENTILE_DISC ( 0.25 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P25
	, PERCENTILE_DISC ( 0.05 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P05
	, CAST(AVG (avgWage) OVER (PARTITION BY immunised_percent) AS DECIMAL(28,2)) AS [Vidutinis atlyginimas]
	, COUNT(avgWage) OVER (PARTITION BY immunised_percent) AS [Viso įmonių/organizacijų]
	, SUM(I.employees) OVER (PARTITION BY immunised_percent) AS [Viso darbuotojų]
	, AVG(I.employees) OVER (PARTITION BY immunised_percent) AS [Vidutinis kompanijų dydis]
	FROM [monthly-2021-cleaned] W INNER JOIN Imunizacija_pagal_darbovietes I ON W.jarCode = I.workplace_code
	WHERE [MONTH] = 202105
	)
SELECT 
	WS.*
FROM WageStats WS
	ORDER BY WS.[Imunizacijos procentas] DESC
```

Rezultate gauname tokius rezultatus:
![Altyginimų vidurkiai pagal pasiskiepijamo procentą įmonėse](/img/stats20210724.png "Altyginimų vidurkiai pagal pasiskiepijamo procentą įmonėse")

Taip pat galime pasižiūrėti statistiką be Vilniaus, panaudojus šią užklausą:

```sql
WITH WageStats AS 
(SELECT DISTINCT immunised_percent AS [Imunizacijos procentas]
	, MAX(avgWage) OVER (PARTITION BY immunised_percent) [Max Vidutinis atlyginimas]
	, PERCENTILE_DISC ( 0.99 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P99
	, PERCENTILE_DISC ( 0.95 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P95
	, PERCENTILE_DISC ( 0.75 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P75
	, PERCENTILE_DISC ( 0.5 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P50 
	, PERCENTILE_DISC ( 0.25 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P25
	, PERCENTILE_DISC ( 0.05 ) WITHIN GROUP ( ORDER BY avgWage ASC ) OVER (PARTITION BY immunised_percent) AS P05
	, CAST(AVG (avgWage) OVER (PARTITION BY immunised_percent) AS DECIMAL(28,2)) AS [Vidutinis atlyginimas]
	, COUNT(avgWage) OVER (PARTITION BY immunised_percent) AS [Viso įmonių/organizacijų]
	, SUM(I.employees) OVER (PARTITION BY immunised_percent) AS [Viso darbuotojų]
	, AVG(I.employees) OVER (PARTITION BY immunised_percent) AS [Vidutinis kompanijų dydis]
	FROM [monthly-2021-cleaned] W INNER JOIN Imunizacija_pagal_darbovietes I ON W.jarCode = I.workplace_code
	WHERE [MONTH] = 202105 AND I.municipality != 'Vilniaus m. sav.'
	)
SELECT 
	WS.*
FROM WageStats WS
	ORDER BY WS.[Imunizacijos procentas] DESC
```
![Altyginimų vidurkiai pagal imunizacijos procentą įmonėse (be Vilniaus m. sav.)](/img/stats20210724-excludingVilnius.png "Altyginimų vidurkiai pagal imunizacijos procentą įmonėse (be Vilniaus m. sav.)")

---
Panaudoti įrankiai:
* [SQL Server 2019 Express edition](https://www.microsoft.com/en-us/Download/details.aspx?id=101064)
* [SQL Server Management studio](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15)
