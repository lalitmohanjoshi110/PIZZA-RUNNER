# :pizza: Case Study #2 - Pizza Runner

## 📝 Solution C. Ingredient Optimization

*In this section, I first got my hands dirty with solving each query in bits and pieces. Then, I realised there are certain results(tables) which are repeatedly used in many queries. So, I created them as temporary tables.*
  
*While solving the queries, I also realised that data type of certain TEXT columns needs to be converted to Character type (Nvarchar) so that the various String Functions (like STRING_SPLIT(), STRING_AGG(), and CONCAT_WS()) can be performed on them.*

### Required Data Type Transformations 

- 1. Creating a temporary table `#pizza_recipes` and changing the data type of `toppings` from **TEXT** to **NVARCHAR(100)** to be able to perform STRING_SPLIT() on it.

````sql
SELECT pizza_id , toppings
INTO #pizza_recipes
FROM pizza_recipes;

ALTER TABLE #pizza_recipes
ALTER COLUMN toppings NVARCHAR(100);
````
- 2. Creating a temporary table `#pizza_toppings` and changing the data type of `topping_name` from **TEXT** to **NVARCHAR(100)** to be able to perform STRING_AGG() on it.

````sql
SELECT topping_id,topping_name
INTO #pizza_toppings
FROM pizza_toppings;

ALTER TABLE #pizza_toppings
ALTER COLUMN topping_name NVARCHAR(100);
````
- 3. Creating a temporary table `#pizza_names` and changing the data type of `pizza_name` from **TEXT** to **NVARCHAR(100)** to be able to perform CONCAT_WS() on it.

````sql
SELECT * 
INTO #pizza_names
FROM pizza_names;

ALTER TABLE #pizza_names
ALTER COLUMN pizza_name NVARCHAR(100);
````

*** 
*We need to create a few temporary tables with more simple and clear formats using the given tables, so that we can easily answer the given questions.*

### Required Temporary tables 

**1.**  Creating the `#pizzas_with_toppings` table (*It contains pizza ids with separated topping ids and separated topping names with one row for each topping name*).
- STRING_SPLIT() is a user_defined table_valued function that splits a string array into multiple rows of substrings based on the specified separator.
- Input_string (1st argument for this function) should be of the type NVARCHAR, VARCHAR, NCHAR or CHAR.

````sql
SELECT	
 r.pizza_id,TRIM(t.value) AS topping_id,pt.topping_name  
--TRIM() removes any leading or trailing spaces and makes sure the substrings are aligned properly.
INTO #pizzas_with_toppings
FROM #pizza_recipes as r
CROSS APPLY STRING_SPLIT(r.toppings, ',') as t
JOIN #pizza_toppings pt
ON TRIM(t.value) = pt.topping_id;
````
**Before:**

|pizza_recipes|pizza_toppings|
|---|---|
|![Screenshot 2022-12-31 140551](https://user-images.githubusercontent.com/96012488/210130640-70a025ec-e642-4831-a49c-5b08d0c8645d.png)|![Screenshot 2022-12-31 140635](https://user-images.githubusercontent.com/96012488/210130657-104f4396-0d93-4a6f-8461-31ab08e558ac.png)|

**After:   `pizzas_with_toppings`**

![Screenshot 2022-12-31 140130](https://user-images.githubusercontent.com/96012488/210130552-f9b319c6-5aa2-4182-866c-4d541d10da44.png)

**2.**  Adding a column `record_id` to the table `#customer_orders` to select each pizza in an order individually, using IDENTITY() function.
- IDENTITY() is a function used to create auto-incrementing columns, i.e., the value of the column increases automatically everytime a new row is inserted into the table.
- It has two optional arguments Seed and Step, default values for both is 1.

````sql
ALTER TABLE #customer_orders
ADD record_id INT IDENTITY(1,1)
````
| Before|After|
|---|---|
|![Screenshot 2022-12-31 141736](https://user-images.githubusercontent.com/96012488/210130906-5a5d2ffc-5b51-4841-a827-3d3231a68564.png)|![Screenshot 2022-12-31 141930](https://user-images.githubusercontent.com/96012488/210130942-7d14e085-b252-427a-9ddc-6d44eeb52797.png)|



**3.** Creating the table `records_with_exclusions` (*It contains record_ids and exclusions for each record*).

````sql
SELECT 
 record_id, value AS exclusions
INTO #records_with_exclusions
FROM #customer_orders c
CROSS APPLY STRING_SPLIT(c.exclusions,',') 
WHERE exclusions != '';
````

![Screenshot 2022-12-31 142124](https://user-images.githubusercontent.com/96012488/210130980-c378d835-cf1a-4b11-b0ca-7dcf5bbcfca5.png)


**4.**  Creating the table` records_with_extras` (*It contains record_ids and extras for each record*)

````sql
SELECT 
 record_id, value AS extras
INTO #records_with_extras
FROM #customer_orders c
CROSS APPLY STRING_SPLIT(c.extras,',') 
WHERE extras != '';
````

![Screenshot 2022-12-31 142209](https://user-images.githubusercontent.com/96012488/210130991-f06f5273-ed24-41d8-bdaf-5ca702122673.png)

***

***Let's answer the Questions now!***

#### 1. What are the standard ingredients for each pizza?

**Approach:**
- We will use the newly created temporary table #pizzasa_with_toppings which contains pizza ids with separated topping ids and separated topping names.
- Using this table, we will retrieve a string array consisting of topping names for each of the Pizza id.
- To create a string array of topping names, we need to perform STRING_AGG() on it.
- STRING_AGG() is a user-defined function that concatenates rows of strings into a single string, separated by a specified separator.

````sql
SELECT 
 pizza_id, STRING_AGG(topping_name,', ') AS Toppings
FROM #pizzas_with_toppings
GROUP BY pizza_id;
````

**Answer:**

![Screenshot 2022-12-31 142652](https://user-images.githubusercontent.com/96012488/210131121-00cfd5d9-85c5-4a84-aed7-08196c004ffa.png)


#### 2. What was the most commonly added extra?  

**Approach:**
- We will use the temporary table #records_with_extras and join it with #pizza_toppings for topping_name.

````sql
SELECT 
 pt.topping_name, COUNT(r.record_id) AS number_of_addition
FROM #records_with_extras r
JOIN #pizza_toppings pt
ON r.extras = pt.topping_id
GROUP BY pt.topping_name;
````
**Answer:**

![Screenshot 2022-12-31 142804](https://user-images.githubusercontent.com/96012488/210131142-d6bd8fbe-0740-490d-af51-3094b6fbff7f.png)

- Bacon was the most commonly added extra.

#### 3. What was the most common exclusion?  

**Approach:**
- We will use the temporary table #records_with_exclusions and join it with #pizza_topping for topping_name.

````sql
SELECT 
 pt.topping_name, COUNT(r.record_id) AS number_of_removals
FROM #records_with_exclusions r
JOIN #pizza_toppings pt
ON r.exclusions = pt.topping_id
GROUP BY pt.topping_name;
````

**Answer:**

![Screenshot 2022-12-31 142838](https://user-images.githubusercontent.com/96012488/210131160-a796435c-0583-4bb7-9572-50f42de2a621.png)

- Cheese was the most common exclusion.



### Learnings  

-  Various new STRING Manipulation Functions like STRING_SPLIT(), STRING_AGG()
-  The tables provided were not in the format that you can directly fetch the results from. I had to create many new tables, by combining the existing tables, in the desired formats to get the desired results.

***Click [here]() for solution for D. Pricing and Ratings!***
