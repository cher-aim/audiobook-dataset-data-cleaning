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

### 1. Bronze Stage: Data Quality Assurance
> The initial phase of our data cleaning process, termed the "Bronze Stage", is dedicated to assessing the overall quality of the dataset. This stage is crucial for identifying fundamental issues that could affect subsequent analyses, including missing values, duplicates, and data integrity.

**1. Missing Values Check**
> To ensure completeness of the dataset, we first conducted a comprehensive search for missing entries across all columns.
```sql
- SQL Query to identify any records with missing values across all columns
SELECT *
FROM `cher-pre-project.audible.adb`
WHERE COALESCE(name, author, narrator, time, releasedate, language, stars, price, ratings) IS NULL;
```
**Results:**
The dataset exhibits a complete set of data with no missing values, indicating high-quality initial data capture.

**2. Verifying Data Temporal Integrity and Uniqueness**
> We ensure the dataset's relevance to the defined timeframe and the uniqueness of records to avoid skewed insights from redundant data.
```sql
-- SQL Query to validate the release dates fall within the expected range (1998 to 2025)
SELECT *
FROM `cher-pre-project.audible.adb`
WHERE EXTRACT(YEAR FROM releasedate) < 1998 OR EXTRACT(YEAR FROM releasedate) > 2025;
```
**Results:**
All entries fall within the expected temporal range, confirming the dataset's contextual validity.

```sql
-- SQL Query to detect any duplicate records based on key attributes
SELECT *
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY name, author, narrator, time, releasedate, language, stars, price ORDER BY name) AS row_num
  FROM `cher-pre-project.audible.adb`
) AS subq
WHERE row_num > 1;
```

**Results:**
No duplicate records were found, affirming the dataset's integrity in this regard.

**3. Validation of Author and Narrator Prefixes**
> Given the structured nature of the author and narrator fields, we verified the uniformity of prefixes ('Written by:' and 'Narrated by:') to ensure consistent data entry.
```sql
SELECT COUNT(*) AS total_rows,
       SUM(CASE WHEN author LIKE 'Writtenby:%' THEN 1 ELSE 0 END) AS matching_at_rows,
       SUM(CASE WHEN narrator LIKE 'Narratedby:%' THEN 1 ELSE 0 END) AS matching_nt_rows
FROM `cher-pre-project.audible.adb`;
```

**Results:** 
All entries correctly included the specified prefixes, confirming data consistency in textual metadata.

**4. Time Column Consistency Check**
```sql
-- Review 'time' column format for conversion readiness
SELECT time
FROM `cher-pre-project.audible.adb`;
```
**Results:**  
The diverse formats identified (e.g., '1 hr and 6 mins', '2 hrs and 16 mins') necessitate a comprehensive conversion strategy to facilitate quantitative analysis.

**5. Stars Column Format and Ratings Separation**
> Our analysis of the stars column aimed to distinguish between actual ratings and the textual representation of unrated items, setting the stage for a structured conversion into numerical data.
```sql
-- Analyze 'stars' column to prepare for splitting into numeric ratings and counts
SELECT stars, COUNT(stars) as stars_count
FROM `cher-pre-project.audible.adb`
GROUP BY stars;
```
**Results:** 
The dataset contains ratings in two main formats: numeric ratings (e.g., '4.5 out of 5 stars') and 'Not rated yet'. This insight guides the separation of ratings into quantifiable metrics.

### 2. Silver Stage: Format Normalization and Data Enrichment
> In the Silver Stage, we focus on refining the dataset by transforming formats and qualitative values to ensure consistency and enhance data usability. This stage is pivotal in preparing the data for in-depth analysis by standardizing textual formats and resolving qualitative discrepancies.

**1. Standardizing Author and Narrator Fields**
> Cleansing the **'author'** and **'narrator'** fields of predefined prefixed, which, while informative, are unnecessary for our analysis and could introducr inconsistency in data processing.
```sql
-- Remove 'Writtenby:' prefix from 'author' and 'Narratedby:' from 'narrator' for uniformity
SELECT 
  name,
  REPLACE(author, 'Writtenby:', '') AS author_cleaned,
  REPLACE(narrator, 'Narratedby:', '') AS narrator_cleaned,
  time,
  releasedate,
  language,
  stars,
  price,
  ratings
FROM `cher-pre-project.audible.adb`;
```
**Explanation:**
> The REPLACE function is employed to eliminate the prefixes, ensuring that both author and narrator fields are standardized, focusing solely on the names, which are critical for any relational or comparative analysis.

**double check**
```sql
SELECT author, narrator
FROM `cher-pre-project.audible.adb`
WHERE UPPER(author) LIKE 'WRITT%' OR UPPER(narrator) LIKE 'NAR%';
--Clear!
```
**Explanation:** 
> This query confirms the efficacy of the prefix removal, using the UPPER function to avoid case sensitivity issues, ensuring that our data transformation has been thoroughly applied.

**2. Language Column Consistency Adjustment**
> Given the importance of linguistic analysis in audiobooks, ensuring a consistent format in the language column is imperative for accurate categorization and subsequent analysis.
```sql
-- Standardize the format of the 'language' column to capitalize only the first letter
SELECT 
  name,
  author_cleaned,
  narrator_cleaned,
  time,
  releasedate,
  CONCAT(UPPER(SUBSTRING(language, 1, 1)), LOWER(SUBSTRING(language, 2))) AS language_standardized,
  stars,
  price,
  ratings
FROM `cher-pre-project.audible.adb`;
```
**Explanation:**
The **`CONCAT`**, **`UPPER`**, and **`LOWER`** functions work together to standardize the language field, ensuring that only the first letter is capitalized. 

> Through the Silver Stage's transformations, we achieve a refined dataset with enhanced consistency and clarity. This stage underscores our meticulous approach to data preparation, laying a solid foundation for the Gold Stage, where we will focus on quantitative transformations and optimization for in-depth analysis.

### 3. Gold Stage: Quantitative Transformation for In-Depth Analysis
> The Gold Stage is where we fine-tune the dataset for deep analysis, focusing on converting text to numbers and ensuring all data is in a format ready for statistical analysis.

**Full Code**
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
### Code Breakdown
**1. Converting Time into Minutes**
> We convert the time data to minutes. This uniformity in the time field is essential for any temporal analysis, such as calculating average listening durations or comparing lengths across audiobooks.
```sql
-- Transform 'time' column into a consistent numeric format (minutes)
WITH gold_stage AS (
  SELECT 
    name,
    author,
    narrator,
    CASE
      WHEN LOWER(time) LIKE '%hr%' AND LOWER(time) LIKE '%min%' THEN
        CAST(REGEXP_EXTRACT(time, r'(\d+) hr') AS INT64) * 60 +
        CAST(REGEXP_EXTRACT(time, r'(\d+) min') AS INT64)
      WHEN LOWER(time) LIKE '%hr%' THEN
        CAST(REGEXP_EXTRACT(time, r'(\d+) hr') AS INT64) * 60
      WHEN LOWER(time) LIKE '%min%' THEN
        CAST(REGEXP_EXTRACT(time, r'(\d+) min') AS INT64)
      ELSE
        CAST(REGEXP_EXTRACT(time, r'(\d+)') AS INT64)
    END AS duration_minutes
  FROM `cher-pre-project.audible.adb`
)
```
**Explanation:**
Regular expressions **`(REGEXP_EXTRACT)`** parse complex string patterns, extracting numeric values that represent hours and minutes, which we then convert to a single unit (minutes). This uniformity allows for direct comparisons and aggregations.

**2. Refining the Stars and Ratings Columns**
> We split the stars column into numeric ratings and the number of ratings, turning words into actionable data.
```sql
-- Extract numeric values for 'stars' and 'ratings'
SELECT 
  *,
  CASE 
    WHEN LOWER(stars) LIKE 'not%' THEN NULL
    ELSE CAST(REGEXP_EXTRACT(stars, r'(\d+\.\d+|\d+) out of 5 stars') AS FLOAT64)
  END AS stars_numeric,
  CASE 
    WHEN stars LIKE '%rating' THEN CAST(REGEXP_EXTRACT(stars, r'(\d+) rating') AS INT64)
    ELSE CAST(REGEXP_EXTRACT(stars, r'(\d+) ratings') AS INT64)
  END AS ratings_count
FROM gold_stage
```
**Explanation:** 
Using **`REGEXP_EXTRACT`** allows us to parse and separate the numerical rating and review count from a mixed text field. Converting these to numeric formats (**`FLOAT64`** for ratings and **`INT64`** for counts) enables accurate mathematical operations, like averages or sums, providing clear metrics for analysis.

**3. Normalizing Price**
> Accurate financial analysis requires numerical price values, including converting textual representations of free items to zero.

```sql
-- Convert 'price' to a consistent numeric format, handling 'Free'
SELECT 
  *,
  CASE
    WHEN LOWER(price) LIKE 'fr%' THEN 0
    ELSE CAST(REPLACE(price, ',', '') AS FLOAT64)
  END AS price_numeric
FROM gold_stage
```
**Explanation:**
The **`CASE`** statement distinguishes between 'Free' and numeric prices, while **`REPLACE`** and **`CAST`** functions remove commas for thousands and convert the cleaned string to a **`FLOAT64`**. This transformation is crucial for financial analytics, allowing us to perform calculations such as total revenue or average price.

> In the Gold Stage, our transformations are guided by the need for a dataset that supports robust statistical analysis. By converting text to numeric formats, we ensure our dataset is ready for algorithms that require quantitative input. The choice of functions like REGEXP_EXTRACT, CAST, and REPLACE is motivated by their ability to parse, standardize, and convert data, preparing it for meaningful analysis and insights.

# Conclusion
> [!NOTE]
> Through the execution of the **Bronze**, **Silver**, and **Gold stages**, we have transformed the Audiobook dataset from a raw collection of diverse and unstructured data points into a refined, structured, and analysis-ready resource. This project involved comprehensive data quality checks, format standardization, and the conversion of qualitative descriptors into quantitative metrics, each step motivated by the goal of enhancing the dataset's utility for in-depth analysis
