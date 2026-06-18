# Amazon Sales Dataset Cleaning and Analysis - SQL Server Project

This project involves cleaning and analyzing an Amazon sales dataset sourced from Kaggle using SQL Server. The dataset contains product listings across multiple electronics categories including USB cables, smart TVs, and WiFi adapters.

## Project overview:
This project aims to improve data quality by applying structured data cleaning techniques to a raw and inconsistent dataset, converting it into a reliable and analysis-ready format. The cleaned dataset is then analyzed to extract meaningful insights, evaluate trends, and answer relevant business questions.

## Dataset Description:
the datsset contains 1466 rows of sales records.

### Data Columns:
product_id, product_name, category, discounted_price, actual_price, discount_percentage, rating, rating_count, about_product, user_id, user_name, review_id, review_title, review_content, img_link, product_link.

## Tools used:
Microsoft SQL Server (SSMS)

## Problems Found in Raw Data:

1. Duplicate rows.
2. standarization of rating, rating_count, and category.
3. change the columns to the right data type.

## SQL Concepts Practiced
1. ALTER TABLE / ALTER COLUMN.
2. UPDATE with SUBSTRING().
3. TRY_CAST.
4. ROW_NUMBER() for deduplication.
5. PARTITION BY with window functions.
6. CTEs (WITH clause).
7. NULL handling.
8. CASE statments.
9. Joins.
10. Aggregate Functions.
11. Filtering Data.
12. Grouping Data.
13. Sorting Results.
14. Top-N Analysis.
15. Distinct Value Analysis.
16. Conditional Logic.
17. Data Categorization / Bucketing.
18. Business Metrics Calculation.
19. Ranking and Comparative Analysis.


## Cleaning Steps Performed
### 1. Dropping the text columns from the table since they are unnecesaary for analysis
#### 1.1 Create a new table
```sql
CREATE TABLE [dbo].[amazon_sales_new](
	[product_id] [nvarchar](255) NULL,
	[product_name] [nvarchar](255) NULL,
	[category] [nvarchar](255) NULL,
	[discounted_price] [nvarchar](255) NULL,
	[actual_price] [nvarchar](255) NULL,
	[discount_percentage] [nvarchar](255) NULL,
	[rating] [float] NULL,
	[rating_count] [nvarchar](255) NULL,
	[about_product] [nvarchar](max) NULL,
	[user_id] [nvarchar](255) NULL,
	[user_name] [nvarchar](255) NULL,
	[review_id] [nvarchar](255) NULL,
	[review_title] [nvarchar](255) NULL,
	[review_content] [nvarchar](max) NULL,
	[img_link] [nvarchar](255) NULL,
	[product_link] [nvarchar](255) NULL
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
GO
```

#### 1.2 Insertion of old data into the new table
```sql
INSERT INTO amazon_sales_new
SELECT *
FROM amazon_sales
;
```

#### 1.3 Dropping the text columns from the table 
```sql
ALTER TABLE amazon_sales_new
DROP COLUMN about_product
;

ALTER TABLE amazon_sales_new
DROP COLUMN user_id
;

ALTER TABLE amazon_sales_new
DROP COLUMN user_name
;

ALTER TABLE amazon_sales_new
DROP COLUMN review_id
;

ALTER TABLE amazon_sales_new
DROP COLUMN review_title
;

ALTER TABLE amazon_sales_new
DROP COLUMN review_content
;

ALTER TABLE amazon_sales_new
DROP COLUMN img_link
;

ALTER TABLE amazon_sales_new
DROP COLUMN product_link
;
```

### 2.Removal of duplicate values 
#### 2.1 Create a new table to insert row_num column
```sql
CREATE TABLE [dbo].[amazon_sales_new_1](
	[product_id] [nvarchar](255) NULL,
	[product_name] [nvarchar](255) NULL,
	[category] [nvarchar](255) NULL,
	[discounted_price] [nvarchar](255) NULL,
	[actual_price] [nvarchar](255) NULL,
	[discount_percentage] [nvarchar](255) NULL,
	[rating] [float] NULL,
	[rating_count] [nvarchar](255) NULL,
	[row_num] [INT] NULL
) ON [PRIMARY]
GO
;
```

#### 2.2 Insert the same data into the new table in addition to partition by all columns
```sql
INSERT INTO amazon_sales_new_1
SELECT	
		*,
		ROW_NUMBER() OVER(
							PARTITION BY	
										product_id, 
										product_name, 
										category, 
										discounted_price, 
										actual_price, 
										discount_percentage, 
										rating, 
										rating_count 
										ORDER BY product_id
							) AS row_num
FROM amazon_sales_new
;
```

#### 2.3 Delete rows where row_num > 1
```sql
DELETE FROM amazon_sales_new_1
WHERE row_num > 1
;
```

### 3. Rating_count standaraization 
#### 3.1 Adding a new column
```sql
ALTER TABLE amazon_sales_new_1
ADD rating_count_clean DECIMAL(10,2)
;
```

#### 3.2 Set the new column to equal the right data type and format wanted 

```sql
UPDATE amazon_sales_new_1
SET rating_count_clean = TRY_CAST(REPLACE(rating_count,',','') AS DECIMAL(10,2))
;
```

#### 3.3 Drop the column of the old data
```sql
ALTER TABLE amazon_sales_new_1
DROP COLUMN rating_count
;
```
#### 3.4 Convert the null value of rating_count to zero 
```sql
UPDATE amazon_sales_new_1
SET rating_count_clean = 0
WHERE rating_count_clean IS NULL
;
```


### 4.Removal of duplicate values for product_id with more than rating_count
#### 4.1 make the max rating_count only exist 
```sql
WITH cte AS 
(
SELECT 
	product_id,
	rating_count_clean
FROM 
(
SELECT 
	product_id,
	rating_count_clean,
	ROW_NUMBER()OVER(PARTITION BY product_id ORDER BY rating_count_clean DESC) AS ord
FROM amazon_sales_new_1
) AS new_table
WHERE ord = 1
)
UPDATE e1
SET e1.rating_count_clean = e2.rating_count_clean
FROM amazon_sales_new_1 AS e1
LEFT JOIN cte AS e2
ON e1.product_id = e2.product_id
;
```
#### 4.2 Create a new table to insert row_num column 
```sql
CREATE TABLE [dbo].[amazon_sales_new_3](
	[product_id] [nvarchar](255) NULL,
	[product_name] [nvarchar](255) NULL,
	[category] [nvarchar](255) NULL,
	[discounted_price] [nvarchar](255) NULL,
	[actual_price] [nvarchar](255) NULL,
	[discount_percentage] [nvarchar](255) NULL,
	[rating] [float] NULL,
	[row_num] [int] NULL,
	[rating_count_clean] [decimal](10, 2) NULL,
	[ord] [INT] NULL
) ON [PRIMARY]
GO
;
```
#### 4.3 Insert into the new table the same data as the old one in addition to partition by product_id
```sql
INSERT INTO [amazon_sales_new_3]
SELECT 
	*,
ROW_NUMBER()OVER(PARTITION BY product_id ORDER BY rating_count_clean DESC) AS ord
FROM amazon_sales_new_1
;
```
#### 4.4 Drop columns where ord is greater than one 
```sql
DELETE FROM amazon_sales_new_3
WHERE ord > 1
;
```

### 5.Discounted_price and actual _price columns standaraization
#### 5.1 Adding a new column for discounted price
```sql
ALTER TABLE amazon_sales_new_3
ADD discounted_price_clean DECIMAL(10,2)
;
```
#### 5.2 Set the new column to equal standarized version of the old column 
```sql
UPDATE amazon_sales_new_3
SET discounted_price_clean = 
	TRY_CAST (REPLACE(SUBSTRING(discounted_price, 2, LEN(discounted_price)),',','') AS DECIMAL(10,2))
;
```
#### 5.3 Adding a new column for actual price
```sql
ALTER TABLE amazon_sales_new_3
ADD actual_price_clean DECIMAL(10,2)
;
```
#### 5.4 Setting the new column to equal standarized version of the old column
```sql
UPDATE amazon_sales_new_3
SET actual_price_clean = 
				TRY_CAST (REPLACE(SUBSTRING(actual_price, 2, LEN(actual_price)),',','') AS DECIMAL(10,2))
;
```
#### 5.5 Drop the past old columns 
```sql
ALTER TABLE amazon_sales_new_3
DROP COLUMN discounted_price
;

ALTER TABLE aءmazon_sales_new_3
DROP COLUMN actual_price
;
```
### 6.Discount_percentage standaraization 
#### 6.1 Adding a new column for discount_percentage
```sql
ALTER TABLE amazon_sales_new_3
ADD discount_percentage_clean INT
;
```
#### 6.2 Updating the old column to equal the new one
```sql
UPDATE amazon_sales_new_3
SET discount_percentage_clean = 
		CAST(ROUND((actual_price_clean - discounted_price_clean) / actual_price_clean * 100, 0) AS INT)
	;
```
#### 6.3 Drop the old column
```sql
ALTER TABLE amazon_sales_new_3
DROP COLUMN discount_percentage
;
```

### 7. Rating standarization 
#### 7.1 adding a new column 
```sql
ALTER TABLE amazon_sales_new_3
ADD rating_clean DECIMAL(10,1)
;
```

#### 7.2 Updating the new column to equal the new version 
```sql
UPDATE amazon_sales_new_3
SET rating_clean = TRY_CAST(rating AS DECIMAL(10,1))
;
```
#### 7.3 Dropping the old column
```sql
ALTER TABLE amazon_sales_new_3
DROP COLUMN rating
;
```

### 8. Category standarization 
```sql
UPDATE amazon_sales_new_3
SET category = SUBSTRING(category, 1, CHARINDEX('|', category) - 1)
;
```

### 9. Drop the unnecessary columns
```sql
ALTER TABLE amazon_sales_new_3
DROP COLUMN row_num
;

ALTER TABLE amazon_sales_new_3
DROP COLUMN ord
;
```

## Analysis Steps Performed
### Q1: What is the total number of products in the dataset?
```sql
SELECT 
	COUNT(product_id) AS total_product_count
FROM amazon_sales_new_3
;
```
### Q2:How many unique categories are there?
```sql
SELECT 
	COUNT(DISTINCT category) AS unique_category_count
FROM amazon_sales_new_3
;
```

### Q3: What is the average rating across all products?
```sql
SELECT 
	AVG(rating_clean) AS avg_rating
FROM amazon_sales_new_3
;
```

### Q4: Which product has the highest discounted price?
```sql
SELECT TOP 1 product_name, discounted_price_clean
FROM amazon_sales_new_3
ORDER BY discounted_price_clean DESC;
```

### Q5: Which product has the lowest actual price?
```sql
SELECT TOP 1 product_name, actual_price_clean
FROM amazon_sales_new_3
ORDER BY actual_price_clean ASC;
```

### Q6: How many products have a discount percentage greater than 50%?
```sql
SELECT 
	COUNT(*) AS count_products_with_discout_per_greater_than_50
FROM amazon_sales_new_3
WHERE discount_percentage_clean > 50
;
```

### Q7: What are the top 10 highest rated products?
```sql
SELECT 
	TOP 10 product_id,
	rating_clean
FROM amazon_sales_new_3
ORDER BY rating_clean DESC
;
```

### Q8: Which category has the highest average discount percentage?
```sql
SELECT 
	TOP 1 
	category,
	AVG(discount_percentage_clean) AS avg_discount_percentage
FROM amazon_sales_new_3
GROUP BY category
ORDER BY avg_discount_percentage DESC
;
```

### Q9: What is the average discount percentage per category?
```sql
SELECT 
	category,
	AVG(discount_percentage_clean) AS avg_discount_percentage
FROM amazon_sales_new_3
GROUP BY category
;
```

### Q10: Which category has the most products listed?

```sql
SELECT 
	TOP 1 category,
	COUNT(*) AS total_product_listed 
FROM amazon_sales_new_3
GROUP BY category
ORDER BY total_product_listed  DESC
;
```

### Q11: What is the average rating per category?

```sql
SELECT 
	category,
	AVG(rating_clean) AS avg_rating
FROM amazon_sales_new_3
GROUP BY category
;
```

### Q12: Which products have a rating above 4.5 and a discount above 50%?
```sql
SELECT 
	product_id,
	product_name
FROM amazon_sales_new_3
WHERE 
rating_clean > 4.5 AND discount_percentage_clean > 50
;
```

### Q13: What is the correlation between discount percentage and rating?
```SQL
SELECT 
    CASE 
        WHEN discount_percentage_clean < 20 THEN '0-20%'
        WHEN discount_percentage_clean < 40 THEN '20-40%'
        WHEN discount_percentage_clean < 60 THEN '40-60%'
        WHEN discount_percentage_clean < 80 THEN '60-80%'
        ELSE '80-100%'
    END AS discount_bucket,
    AVG(rating_clean) AS avg_rating,
    COUNT(*) AS num_products
FROM amazon_sales_new_3
GROUP BY 
    CASE 
        WHEN discount_percentage_clean < 20 THEN '0-20%'
        WHEN discount_percentage_clean < 40 THEN '20-40%'
        WHEN discount_percentage_clean < 60 THEN '40-60%'
        WHEN discount_percentage_clean < 80 THEN '60-80%'
        ELSE '80-100%'
    END
ORDER BY discount_bucket
;
```

### Q14: Which category generates the most potential revenue based on actual price
```SQL
SELECT 
	SUM((actual_price_clean * rating_count_clean))  AS revnue,
	category
FROM amazon_sales_new_3
GROUP BY category
ORDER BY revnue DESC
;
```


