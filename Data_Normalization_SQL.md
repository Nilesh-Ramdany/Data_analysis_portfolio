## Data Normalization

**Aim**: in this project we have data that is in denormalized form and we want to bring it to the third normal form. This allows us to eliminate data redundancy and undesirable characteristics like Insertion, Update and Deletion anomalies.

Consider the following table in our database which contains sales records:

````sql
SELECT *
FROM sales
LIMIT 3;
````

| order_id      | customer_info      | city      | state      | order_date      | products      | category      | list_price      | store_info      | Sales_rep      |
| ------- | ---------- | ------- | --- |-------- | ------- | --------- | ----------- | ------------- | -------- |
| 1      | Johnathan Velazquez, johnathan.velazquez@hotmail.com | Pleasanton | CA  | 2016-01-01  | Electra Townie Original 7D EQ - Women's - 2016,Trek Remedy 29 Carbon Frameset - 2016,Surly Straggler - 2016,Electra Townie Original 7D EQ - 2016,Trek Fuel EX 8 29 - 2016 | Cruisers Bicycles,Mountain Bikes,Cyclocross Bicycles,Cruisers Bicycles,Mountain Bikes | 599.99,1799.99,1549.00,599.99,2899.99 | Santa Cruz Bikes,(831) 476-4321,santacruz@bikes.shop | Mireya Copeland mireya.copeland@bikes.shop |
| 2       | Jaqueline Cummings, jaqueline.cummings@hotmail.com | Huntington Station | NY    | 2016-01-01 | Electra Townie Original 7D EQ - Women's - 2016,Electra Townie Original 7D EQ - 2016 | Cruisers Bicycles,Cruisers Bicycles | 599.99,599.99   | Baldwin Bikes,(516) 379-8888,baldwin@bikes.shop | Marcelene Boyer marcelene.boyer@bikes.shop |
| 3   | Joshua Robertson, joshua.robertson@gmail.com | Patchogue | NY   | 2016-01-02      | Surly Wednesday Frameset - 2016,Electra Townie Original 7D EQ - Women's - 2016 | Mountain Bikes,Cruisers Bicycles | 999.99,599.99 | Baldwin Bikes,(516) 379-8888,baldwin@bikes.shop | Venita Daniel venita.daniel@bikes.shop |

We immediately notice that all the data relating to the sales orders is bundle into a single table (all the information about the products, customer, salesperson, stores are included in the table). 
There is a lot of redundancy. For example, the information about the stores 
(phone, email, address) is repeated in the table. We will address these issues by going through the process of normalization.

### 1NF - First Normal Form
To be in first normal form the data must satisfy the following:
- Every column/attribute should have a single value. Atomicity = 1.
- The first normal form disallows the multi-valued attribute, composite attribute and their combinations.
- Every column should be unique

In the above table, we can see that some of the columns like products and list price contain multiple values (the products purchased in a single order and their list prices).
Also, we have composite attributes such as the customer_info and store_info

we can separate the composite attribute into simple attributes (e.g customer_info into firstname, lastname and email ) using the following:

````sql
Select
	customer_info,
	SUBSTRING_INDEX(customer_info, ' ', 1) firstname,
	SUBSTRING_INDEX(SUBSTRING_INDEX(customer_info, ',', 1), ' ', -1 ) lastname,
	SUBSTRING_INDEX(customer_info, ',', -1) email
From sales
Limit 5;

````


| customer_info      | firstname      | lastname      | email      |
| ------------------ | -------------- |--------------| ---------- |
| Johnathan Velazquez, johnathan.velazquez@hotmail.com | Johnathan      | Velazquez      |  johnathan.velazquez@hotmail.com |
| Jaqueline Cummings, jaqueline.cummings@hotmail.com | Jaqueline      | Cummings       |  jaqueline.cummings@hotmail.com |
| Joshua Robertson, joshua.robertson@gmail.com | Joshua         | Robertson      |  joshua.robertson@gmail.com |
| Nova Hess, nova.hess@msn.com | Nova           | Hess           |  nova.hess@msn.com |
| Arla Ellis, arla.ellis@yahoo.com | Arla           | Ellis          |  arla.ellis@yahoo.com |

To make the change to our table we run the following :
````sql
ALTER TABLE sales
Add customer_firstname Nvarchar(255);

Update sales
SET ustomer_firstname = SUBSTRING_INDEX(customer_info, '  ', 1);


ALTER TABLE sales
Add customer_lastname Nvarchar(255);

Update sales
SET ustomer_lastname = SUBSTRING_INDEX(SUBSTRING_INDEX(customer_info, ',' , 1), '  ', -1 ) lasttname;


ALTER TABLE sales
Add customer_firstname Nvarchar(255);

Update sales
SET customer_email = SUBSTRING_INDEX(customer_info, ',' , -1);
````
Next, we also need to transform each element of products and list_price to a separate row. The rows currently look like this
````sql
Select
	order_id,
	list_price, 
	products
From sales
where order_id=1;
````

| order_id      | list_price      | products      |
| ------------- | --------------- |------------- |
| 1             | 599.99,1799.99,1549.00,599.99,2899.99 | Electra Townie Original 7D EQ - Women's - 2016,Trek Remedy 29 Carbon Frameset - 2016,Surly Straggler - 2016,Electra Townie Original 7D EQ - 2016,Trek Fuel EX 8 29 - 2016 |

To separate the list_price and product names into different rows we will use a recursive query:

````sql
WITH RECURSIVE unwound AS (
    SELECT order_id, list_price, products
     FROM sales
UNION ALL
	SELECT 
		order_id, 
		regexp_replace(list_price, '^[^,]*,', '') list_price,
        regexp_replace(products, '^[^,]*,', '') product_name
	FROM unwound
	-- get the rows that have a comma in list_price
	WHERE list_price LIKE '%,%' and products LIKE '%,%'
  )
	SELECT 
		order_id ,
		regexp_replace(list_price, ',.*', '') list_price,
        regexp_replace(products,  ',.*',', '') product_name
	FROM unwound
    WHERE order_id = 1
    ORDER BY order_id;
````
| order_id      | list_price      | product_name      |
| ------------- | --------------- | ----------------- |
| 1             | 599.99          | Electra Townie Original 7D EQ - Women's - 2016 |
| 1             | 1799.99         | Trek Remedy 29 Carbon Frameset - 2016 |
| 1             | 1549.00         | Surly Straggler - 2016 |
| 1             | 599.99          | Electra Townie Original 7D EQ - 2016 |
| 1             | 2899.99         | Trek Fuel EX 8 29 - 2016 |

Now, we still need the rows to be unique. Products ordered more than once in a single order will cause duplicate rows. Therefore, we will group by order_id and product_name and add a column for quantity to make every row unique (combination of order_id and  product_name is now unique). At this point our table should look like this:

![alt text](https://github.com/Nilesh-Ramdany/Data_analysis_portfolio/blob/main/execution%20plans/1NF.png)

Our table is now in First Normal Form.

### 2NF - Second Normal Form


- The table has be be in First Normal Form
- All non-key attributes must be fully dependent on candidate key. The table should not possess partial dependency. 
+ if a non-key column is partially dependent on the candidate key then we should split them into separate tables
- Every table should have a primary key and relationship between the tables should be formed using the foreign key.

From our table in First Normal Form above, the candidate key is the combination of order_id and product_name. We have partial dependency since for example product category only depends on the product_name.


### 3NF Third Normal Form

- The table should be in Second Normal Form
- There should be no transitive dependency for non-prime attributes. 
+ non-prime-attributes should not depend on other non-prime 
attributes

Applying Second and Third Normal form to the above table results in the following schema:


![alt text](https://github.com/Nilesh-Ramdany/Data_analysis_portfolio/blob/main/execution%20plans/normalized_schema.png)
