# Data Cleaning Process for Audiobook Dataset
 > [!NOTE]
> This uncleaned data is from the Kaggle website. You can find the link for more details here. [https://www.kaggle.com/datasets/snehangsude/audible-dataset](url)

## About Dataset
> This dataset was gathered through web scraping, capturing information reflecting the landscape of the audiobook market. Encompassing details ranging from authors to release dates, the dataset provides valuable insights into the evolution of audiobooks from 1998 to 2025, including pre-planned releases.

**This dataset contains 9 columns:**
- name[STRING]: Name of the audiobook. 
- author[STRING]: Author of the audiobook. 
- narrator[STRING]: Narrator of the audiobook. 
- time[STRING]: Length or duration of the audiobook. 
- releasedate[DATE]: Release date of the audiobook. 
- language[STRING]: Language in which the audiobook is presented.
- stars[STRING]: Number of stars received by the audiobook, reflecting user ratings.
- price[STRING]: Price of the audiobook in Indian Rupees (INR).
- ratings[STRING]: Number of reviews received by the audiobook.

## Data Cleaning Process
To ensure the dataset's reliability and usability, I undertook a series of data cleaning steps. I divided the stage of cleaning into 3 main stages: bronze, silver and gold.

### 1. Bronze Stage
> In bronze stage, I will perform the data quality assessment.

**1. check missing values**
```sql
SELECT *
FROM `cher-pre-project.audible.adb`
WHERE COALESCE(1, 2, 3, 4, 5, 6, 7, 8) IS NULL
```
**Results:**
There is no missing values

**2. check if the data is reasonable**
```sql
SELECT *
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY name, author, narrator, time, releasedate, language, stars, price ORDER BY name) AS row_num
  FROM `cher-pre-project.audible.adb`
) AS subq
WHERE row_num > 1;
```

- Because the data only contains information from 1998 to 2025, we need to check if it corresponds with the context or not.

**Results:**
The data is aligned with the context given.

**3. check duplicate data**
```sql
SELECT *
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY name, author, narrator, time, releasedate, language, stars, price ORDER BY name) AS row_num
  FROM `cher-pre-project.audible.adb`
) AS subq
WHERE row_num > 1;
```

**Results:**
There is no duplicate data

**4. values check**
> Since the 'author' and 'narrator' columns contain the prefixes 'Writtenby:' or 'Narratedby', I need to check if all values contain those prefixes.
```sql
SELECT COUNT(*) AS total_rows,
       SUM(CASE WHEN author LIKE 'Writtenby:%' THEN 1 ELSE 0 END) AS matching_at_rows,
       SUM(CASE WHEN narrator LIKE 'Narratedby:%' THEN 1 ELSE 0 END) AS matching_nt_rows
FROM `cher-pre-project.audible.adb`;
```

**Results:** All of the rows in these columns contain these prefixes; therefore, I will remove them in the silver stage.

**5. check time column**
```sql
SELECT 
    time
FROM  `cher-pre-project.audible.adb`;
```
**Results:**  Values in the time column are in the STRING format (e.g., 1 hr and 6 mins, 2 hrs and 16 mins, 1 hr, or 19 mins). I'll make it consistent by converting them into minutes.

**6. check stars column**
```sql
SELECT
  stars,
  COUNT(stars) as stars_count
FROM `cher-pre-project.audible.adb`
GROUP BY stars; 
```
**Results:** There are 2 formats in the stars column:
    1. Not rated yet
    2. [num] out of 5 stars[num] rating[s] 
Therefore, I need to separate it into 2 columns: stars and ratings. 
- The 'stars' column will contain only the [num] before the words 'out of 5 stars'
- The 'ratings' column will contain only the [num] before the word 'rating(s)'

- Not rated yet will be turned into 'NULL'

### 2. Silver Stage
> This stage will be the transformation of formats and qualitative values.

```sql
SELECT 
  name,
  REPLACE(author, 'Writtenby:', '') AS author,
  REPLACE(narrator, 'Narratedby:', '') AS narrator,
  time,
  releasedate,
  language,
  stars,
  price
FROM `cher-pre-project.audible.adb` ;

--double check
SELECT author, narrator
FROM `cher-pre-project.audible.adb`
WHERE UPPER(author) LIKE 'WRITT%' OR UPPER(narrator) LIKE 'NAR%';
--Clear!

-- 2. Adjust the format of string in 'language' column to ensure consistency
SELECT 
  name,
  author,
  narrator,
  duration_minutes,
  releasedate,
  CONCAT(UPPER(SUBSTR(language, 1, 1)), LOWER(SUBSTR(language, 2))) AS language,
  stars,
  price
FROM `cher-pre-project.audible.adb`;
```

### 3. Gold Stage
> This stage will be the transformation of the quantitative side by prioritizing optimization for analysis.

```sql
WITH gold_stage AS(
  SELECT 
  name,
  author,
  narrator,
  CASE
    WHEN LOWER(time) LIKE '%hr%' AND LOWER(time) LIKE '%min%' THEN
      CAST(REGEXP_EXTRACT(time, r'(\d+) hr') AS INT64) * 60 + CAST(REGEXP_EXTRACT(time, r'(\d+) min') AS INT64)
    WHEN LOWER(time) LIKE '%hr%' THEN
      CAST(REGEXP_EXTRACT(time, r'(\d+) hr') AS INT64) * 60
    WHEN LOWER(time) LIKE '%min%' THEN
      CAST(REGEXP_EXTRACT(time, r'(\d+) min') AS INT64)
    ELSE
      CAST(REGEXP_EXTRACT(time, r'(\d+)') AS INT64)
  END AS duration_minutes, -- convert 'time' column into 'minutes' (int) 
    /*
    if both hr and min patterns are found then hr*60 + min
    if hr only then hr*60
    if min only then min
    */
  releasedate,
  language,
  CASE 
    WHEN LOWER(stars) LIKE 'not%' THEN 0
  ELSE
    CAST(REGEXP_EXTRACT(stars, r'(\d+\.\d+|\d+) out of 5 stars') AS FLOAT64) END AS stars,
  CASE 
    WHEN stars LIKE '%rating' 
    THEN CAST(REGEXP_EXTRACT(stars, r'(\d+) rating') AS INT64) 
    WHEN LOWER(stars) LIKE 'not%' THEN 0
  ELSE
    CAST(REGEXP_EXTRACT(stars, r'(\d+) ratings') AS INT64) 
  END AS ratings,
    /*
    clean 'stars' column
    there are 3 things to consider:
    1. Not rated yet = 0 in both columns
    2. [num] out of 5 stars = 'stars' col (FLOAT64)
    3. It has [num]ratings in the column! = 'rating' col (INT64)
    */
  CASE
    WHEN LOWER(price) LIKE 'fr%' THEN 0
    ELSE CAST(REPLACE(price, ',', '') AS FLOAT64) 
    END AS price
    /* 
    Price column
    if price = free then 0 
    else price and should be 'FLOAT' data type
    */

FROM `cher-pre-project.audible.adb1`
)
SELECT 
  *
FROM gold_stage
```

