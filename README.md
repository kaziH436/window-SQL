# SQL Window Functions Tutorial Notes

**Source:** SQL Window Function | How to write SQL Query using Frame Clause, CUME_DIST | SQL Queries Tutorial (techTFQ)

---

## 1. Table Schema & Sample Dataset

The tutorial uses a sample table named `product` containing 27 records across electronic device categories [2].

### Table Structure: `product`

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `product_category` | VARCHAR | Category of product (Phone, Laptop, Earphone, Headphone, Smartwatch) [2] |
| `brand` | VARCHAR | Manufacturer / Brand name (Apple, Samsung, Sony, etc.) [2, 3] |
| `product_name` | VARCHAR | Specific product name (Airpods Pro, XPS 17, Galaxy Z Fold 3, etc.) [2, 3] |
| `price` | NUMERIC | Retail price of the product [2, 3] |

---

## 2. `FIRST_VALUE()` Window Function

### **Concept & Use Case**
* **Purpose:** Returns a column value from the **first record** within a partition [3].
* **Use Case:** Find the most expensive product within each product category while preserving all rows [3, 6].

### **SQL Query**
```sql
SELECT 
    *,
    FIRST_VALUE(product_name) OVER (
        PARTITION BY product_category 
        ORDER BY price DESC
    ) AS most_expensive_product
FROM product;
```

### **Line-by-Line Explanation**
1. `SELECT *`: Selects all existing columns (`product_category`, `brand`, `product_name`, `price`) from the `product` table [3].
2. `FIRST_VALUE(product_name)`: Specifies the window function and passes `product_name` as the target column to extract from the first row of each partition [4].
3. `OVER (`: Opens the window specification clause required for window functions [4].
4. `PARTITION BY product_category`: Groups rows into distinct partitions by product category [5].
5. `ORDER BY price DESC`: Sorts rows within each partition by price in descending order, placing the highest price at row index 1 [6].
6. `) AS most_expensive_product`: Closes the window definition and aliases the column as `most_expensive_product` [6].
7. `FROM product;`: Identifies `product` as the source table [3].

---

## 3. The Frame Clause & `LAST_VALUE()`

### **Concept & Default Frame Behavior**
* **Purpose:** `LAST_VALUE()` returns a column value from the **last record** within a window frame [8].
* **Default Frame:** By default, when `ORDER BY` is specified inside `OVER()`, SQL applies:
  `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` [13].
* **The Problem:** Because the default frame ends at `CURRENT ROW`, `LAST_VALUE()` evaluates only up to the current row, returning the current row's value rather than the last row of the entire partition [14].
* **The Solution:** Explicitly extend the frame boundary using `UNBOUNDED FOLLOWING` [17].

### **Correct Query (Explicit Frame Clause)**
```sql
SELECT 
    *,
    FIRST_VALUE(product_name) OVER (
        PARTITION BY product_category 
        ORDER BY price DESC
    ) AS most_expensive_product,
    LAST_VALUE(product_name) OVER (
        PARTITION BY product_category 
        ORDER BY price DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS least_expensive_product
FROM product;
```

### **Line-by-Line Explanation**
1. `SELECT *`: Retrieves all standard table columns [3].
2. `FIRST_VALUE(product_name) OVER (...)`: Finds the highest-priced product name in each category [4, 6].
3. `LAST_VALUE(product_name)`: Calls `LAST_VALUE` to fetch `product_name` from the last row of the frame [9].
4. `OVER (`: Opens the window specification clause [9].
5. `PARTITION BY product_category`: Partitions dataset by product category [9].
6. `ORDER BY price DESC`: Orders rows inside each partition by price descending [9].
7. `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`: Explicitly sets the frame clause from the start of the partition (`UNBOUNDED PRECEDING`) to the end (`UNBOUNDED FOLLOWING`) [17, 18].
8. `) AS least_expensive_product`: Aliases the output column as `least_expensive_product` [10].
9. `FROM product;`: Targets the `product` table [3].

### **`ROWS` vs. `RANGE` Comparison**
* **`ROWS`**: Evaluates frame boundaries strictly by physical row counts [18, 20].
* **`RANGE`**: Evaluates frame boundaries logically based on duplicate value equality in the `ORDER BY` column [18, 20, 21].

---

## 4. Reusable Window Definitions (`WINDOW` Clause)

### **Concept & Use Case**
* **Purpose:** Avoids repeating duplicate `OVER (...)` specifications across multiple window functions [23, 25].

### **SQL Query**
```sql
SELECT 
    *,
    FIRST_VALUE(product_name) OVER w AS most_expensive_product,
    LAST_VALUE(product_name) OVER w AS least_expensive_product
FROM product
WINDOW w AS (
    PARTITION BY product_category 
    ORDER BY price DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

### **Line-by-Line Explanation**
1. `SELECT *`: Selects all original columns [23].
2. `FIRST_VALUE(product_name) OVER w AS most_expensive_product`: Applies `FIRST_VALUE` referencing named window `w` [24, 25].
3. `LAST_VALUE(product_name) OVER w AS least_expensive_product`: Applies `LAST_VALUE` referencing named window `w` [24, 25].
4. `FROM product`: Specifies the table source [23].
5. `WINDOW w AS (`: Declares a named, reusable window definition named `w` [24].
6. `PARTITION BY product_category`: Defines category-based partitioning inside `w` [24].
7. `ORDER BY price DESC`: Sorts price descending inside `w` [24].
8. `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`: Sets frame boundaries across the entire partition [26].
9. `);`: Closes the named window definition [24].

---

## 5. `NTH_VALUE()` Window Function

### **Concept & Use Case**
* **Purpose:** Extracts a column value from the **N-th row** of an ordered window frame [27]. Returns `NULL` if that row position does not exist [30].

### **SQL Query**
```sql
SELECT 
    *,
    FIRST_VALUE(product_name) OVER w AS most_expensive_product,
    LAST_VALUE(product_name) OVER w AS least_expensive_product,
    NTH_VALUE(product_name, 2) OVER w AS second_most_expensive_product
FROM product
WINDOW w AS (
    PARTITION BY product_category 
    ORDER BY price DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

### **Line-by-Line Explanation**
1. `NTH_VALUE(product_name, 2)`: Accepts two parameters: target column (`product_name`) and 1-based row index (`2` for 2nd row) [28].
2. `OVER w`: Reuses window specification `w` [28].
3. `AS second_most_expensive_product`: Aliases the resulting column [29].

---

## 6. `NTILE()` Window Function

### **Concept & Use Case**
* **Purpose:** Divides ordered rows into `N` equal (or near-equal) buckets [32, 35].

### **SQL Query**
```sql
SELECT 
    product_name,
    CASE 
        WHEN x.buckets = 1 THEN 'Expensive Phone'
        WHEN x.buckets = 2 THEN 'Mid-Range Phone'
        WHEN x.buckets = 3 THEN 'Cheaper Phone'
    END AS phone_category
FROM (
    SELECT 
        *,
        NTILE(3) OVER (
            ORDER BY price DESC
        ) AS buckets
    FROM product
    WHERE product_category = 'Phone'
) x;
```

### **Line-by-Line Explanation**
1. `SELECT product_name,`: Selects product name in outer query [36].
2. `CASE`: Opens conditional expression to map bucket IDs to category names [36, 37].
3. `WHEN x.buckets = 1 THEN 'Expensive Phone'`: Maps bucket `1` to `'Expensive Phone'` [37].
4. `WHEN x.buckets = 2 THEN 'Mid-Range Phone'`: Maps bucket `2` to `'Mid-Range Phone'` [37].
5. `WHEN x.buckets = 3 THEN 'Cheaper Phone'`: Maps bucket `3` to `'Cheaper Phone'` [37].
6. `END AS phone_category`: Closes CASE expression and labels column `phone_category` [37].
7. `FROM (`: Starts subquery `x` [36].
8. `SELECT *,`: Selects source columns [33].
9. `NTILE(3) OVER (`: Divides rows into 3 buckets [33].
10. `ORDER BY price DESC`: Sorts phones by price descending [34].
11. `) AS buckets`: Aliases output as `buckets` [34].
12. `FROM product`: Source table [33].
13. `WHERE product_category = 'Phone'`: Filters to phone records only [34].
14. `) x;`: Aliases derived table as `x` [36].

---

## 7. `CUME_DIST()` (Cumulative Distribution)

### **Concept & Formula**
* **Purpose:** Calculates cumulative distribution percentage between 0 and 1 [38, 45].
* **Formula:** $\text{CUME\_DIST} = \frac{\text{Rows with value } \le \text{ Current Row Value}}{\text{Total Rows in Partition}}$ [45]

### **SQL Query**
```sql
SELECT 
    product_name,
    CONCAT(ROUND(cume_dist_val * 100, 2), '%') AS cume_dist_percentage
FROM (
    SELECT 
        *,
        CUME_DIST() OVER (
            ORDER BY price DESC
        ) AS cume_dist_val
    FROM product
) x
WHERE x.cume_dist_val <= 0.30;
```

### **Line-by-Line Explanation**
1. `SELECT product_name,`: Selects product name [43].
2. `CONCAT(ROUND(cume_dist_val * 100, 2), '%') AS cume_dist_percentage`: Converts decimal ratio to percentage string [41, 43].
3. `FROM (`: Opens subquery [43].
4. `CUME_DIST() OVER (`: Evaluates cumulative distribution [39].
5. `ORDER BY price DESC`: Orders dataset by price descending [40].
6. `) AS cume_dist_val`: Aliases calculation [40].
7. `FROM product`: Targets `product` table [38].
8. `) x`: Aliases subquery as `x` [44].
9. `WHERE x.cume_dist_val <= 0.30;`: Filters for top 30% cumulative distribution [44].

---

## 8. `PERCENT_RANK()` (Percentage Rank)

### **Concept & Formula**
* **Purpose:** Calculates relative percentage rank from 0 to 1 [47, 49]. First row is always 0 [49].
* **Formula:** $\text{PERCENT\_RANK} = \frac{\text{Current Row Rank} - 1}{\text{Total Partition Rows} - 1}$ [49, 50]

### **SQL Query**
```sql
SELECT 
    product_name,
    price,
    ROUND(
        (PERCENT_RANK() OVER (ORDER BY price ASC) * 100)::numeric, 
        2
    ) AS percentage_rank
FROM product;
```

### **Line-by-Line Explanation**
1. `SELECT product_name, price,`: Retrieves name and price columns [47].
2. `PERCENT_RANK() OVER (`: Computes relative percentage rank [47].
3. `ORDER BY price ASC`: Sorts by price ascending [48].
4. `* 100`: Converts to 0–100 scale [48].
5. `::numeric`: Casts to numeric data type [48].
6. `ROUND(..., 2)`: Rounds output to 2 decimal places [48].
7. `AS percentage_rank`: Aliases output column [48].
8. `FROM product;`: Targets `product` table [47].
