# MySQL Technical Exercise

################## DOCUMENTATION ####################
### Test Setup

For this exercise I will use MySQL 5.6.26 integrated within Xampp's apache distribution.

###'
Database name: tech_ex
Storage Engie: InnoDB
Collation: utf8mb4_unicode_ci
### 

### Tables
Table: "raw_orders"
Collation: "utf8mb4_unicode_ci"
Default Indexes: "order_id" and "PRIMARY"

Table: "formatted_orders"
Collation: "utf8mb4_unicode_ci"
Default Indexes: "client_order_id" and "PRIMARY"


#####################################################
############### QUESTIONS ###########################
1.1.1 - Since the "order_revenue" column, in both tables corresponds to currency, it will need an exact representation and not an approximate data type "FLOAT", therefore
	I will use a "fixed-point" decimal data type. Change "FLOAT" data type to "DECIMAL(15, 2)" data type;
	
1.2.1 - Created and used a "stored procedure" loop for inserting data into the table "raw_orders" (check Logs section 1.2.1 for more details)
	
1.2.2 - Created another "stored procedure" named "populateFormattedOrders" to populate the data from the "formatted_orders" table (see Logs section 1.2.2 for more details)

1.2.3.1 - Collation error between the following fields (see section Logs 1.2.3 or more details):
	"order_id" 			- Collation "utf8mb4_unicode_ci"
	"client_order_id"	- Collation "utf8mb4_general_ci"
	Solution: Collate one of the fields to match the other:
		"client_order_id COLLATE utf8mb4_unicode_ci"
	
1.2.3.2 - Not applicable.
	
1.2.4 - In order to create a foreign key between the two tables columns: "client_order_id" and "order_id", the two columns must have the same collation type.
	Since the example given has different collations, I created a query to alter the collation type of the column "client_order_id" to "utf8mb4_unicode_ci". After
	matching the two collations, it is now possible to create the foreign key "FK_orderID". Since the "DELETE" and "UPDATE" actions were not specified, these were assumed as "NO ACTION".
	(see Logs section 1.2.4 for further details)


-----------------------
	
2.1.1 - The query can be improved by substituting the nested select within the join, by the necessary condition upon the "JOIN" command execution.	

2.1.2 - (NO?!)

2.1.3 - (INDEXING)

-----------------------

3.1 - To achieve this result, I decided to create a function where I would concatenate all the amount of multiples of ten. Since not particular approuch restriction was given
I assumed that the result would have to be a concatenation of all amount of "order_revenue" per value "order_id" interval;
(see Logs section 3.1 for further details)

-----------------------

4.1 - To intersect rows that contain the same ID from both tables, "INNER JOIN". (see Logs section 4.1 for further details)

4.2 - In order get the "value" column for both table, while only searching for an ID in the table "t1", "LEFT JOIN" should be used  with the table "t1" on the left side of the join command; (see Logs section 4.2 for further details)

4.3 - To get only the which have only an ID in the table "t2", a "RIGHT JOIN" should be used with the table "t1" on the left side of the join command;




#### Notes
- Collation - utf8mb4_unicode_ci - unknown collation to the MySQL server.
Solution: 
	1) Create table with collation "utf8";
	2) Collate the created table to "utf8mb4_unicode_ci".
	
###### 'formatted orders'
- Provided query was misusing the command "use" within the "CREATE TABLE" - In order to create the table I had to remove this command;

#### 1.2.1
```sql
CREATE PROCEDURE InsertRandomData_rawOrders(numRows, min, max)
	BEGIN
		DECLARE i INT;
		SET i = 1;
		START TRANSACTION;
		WHILE i <= NumRows DO
			INSERT INTO raw_orders (order_revenue, order_id) VALUES (MinVal + round(RAND() * (MaxVal - MinVal), 2), UUID());
			SET i = i + 1;
		END WHILE;
		COMMIT;
	END
```
	
#CALL
#  InsertRandomData_rawOrders(5000,
#  1,
#  10000)

#### 1.2.2
```sql
	CREATE PROCEDURE `populateFormattedOrders`()
		BEGIN
			INSERT INTO formatted_orders(client_order_id, order_revenue)
			SELECT raw_orders.order_id, raw_orders.order_revenue
				FROM raw_orders;
			COMMIT;
		END
```

#### 1.2.3.1
```sql
SELECT * FROM formatted_orders AS fo 
	JOIN raw_orders AS ro ON ro.order_id = fo.client_order_id COLLATE utf8mb4_unicode_ci
	WHERE fo.order_id BETWEEN 100 AND 199
```

#### 1.2.3.2
- Not applicable.

#### 1.2.4

###### Steps
	1) There is a need to change the length and collation type of the column "client_order_id" in order to allow the "alter" command to execute
	2) Change collation of column "client_order_id" from table "formatted_orders": "utf8mb4_general_ci" to "utf8mb4_unicode_ci"
```sql
 ALTER TABLE formatted_orders 
	CHANGE client_order_id client_order_id VARCHAR(200) 
	CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL
```

###### After the collation type has been change, the length can now be set to the original length - VARCHAR(50);
```sql 
 ALTER TABLE formatted_orders 
	CHANGE client_order_id client_order_id VARCHAR(50) 
	CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL;
```

###### Add Foreign Key
```sql
 ALTER TABLE formatted_orders ADD CONSTRAINT fk_OrderID
 	FOREIGN KEY ( client_order_id ) REFERENCES raw_orders ( order_id )
 	ON DELETE NO ACTION ON UPDATE NO ACTION
```
	
#### 2.1.1
###### Improvements: 
- Remove nested select;
- Add the command "IGNORE INDEX(PRIMARY)" to the select.

#### 2.2.2

#### 2.2.3
 
# SELECT raw_orders.*
# FROM raw_orders
# JOIN formatted_orders_copy AS fo ON fo.client_order_id  = raw_orders.order_id AND fo.order_id < 450
# WHERE raw_orders.order_revenue > 50

#### 3.1

```sql
	CREATE FUNCTION revenueMultiples10 ()
	RETURNS TEXT
	BEGIN

	   DECLARE concatReturn TEXT DEFAULT "";
	   DECLARE currentReturn VARCHAR(20);
	   DECLARE mValue NUMERIC(15, 2);
	   DECLARE currentCount, it, printIT INT;
	   
	   SET mValue = (SELECT MAX(order_revenue) from formatted_orders);
	   SET currentCount = 0, it = 0, printIT = 0;

	   SEARCH: WHILE it <= mValue DO
		 SET it = it + 10;
		 SET printIT = it - 10;
		 SET currentCount  = (SELECT COUNT(order_revenue) FROM formatted_orders where order_revenue >= printIT AND order_revenue < it);     
		 SET currentReturn = concat(concat(concat("(", concat(concat(printIT, "-"), it)), ")"), " = ");
		 SET currentReturn = concat(currentReturn, concat(currentCount, " | "));
		 SET concatReturn = concat(concatReturn, currentReturn);
		
	   END WHILE SEARCH;

	   RETURN concatReturn;

	END
```

#### 4.1
```sql
 SELECT t1.value as value1, t2.value AS value2 FROM t1
	INNER JOIN t2  ON t2.ID = t1.ID
```

#### 4.2
---
```sql
 SELECT t1.value as value1, t2.value AS value2 FROM t1
	LEFT JOIN t2  ON t1.ID = t2.ID
```
###### OR
```sql
 SELECT t1.value as value1, t2.value AS value2 FROM t2
	RIGHT JOIN t1  ON t1.ID = t2.ID
```

#### 4.3
---
```sql
 SELECT t1.value as value1, t2.value AS value2 FROM t1
 RIGHT JOIN t2  ON t1.ID = t2.ID
```

#### 4.4
* In order to filter NULL values on the table **"t2"** for values in the table **"t1"**, a **"LEFT JOIN"** must be used where the table **"t1"** will be on the left side of the **"JOIN"** command;

* In the second select **"significant_value"**, a **"CASE THEN"** command must be used to choose which value to return.
---
```sql
 SELECT t1.t_id AS product_id, 
	IF(t2.value IS NULL,  t1.value, t2.value) significant_value FROM t1
	LEFT JOIN t2 ON t2.id = t1.t_id
```	

#### 5.1
* Approuch taken:

	1. In order to have a subcategory hierarchy type, a foreign key must be defined so that there is a link between the category **"ID"** and **"parentid"**; 
	
	2. To filter wrong data insertion of the subcategory level, i.e. between 0 and 5, the **"BEFORE INSERT"**
	and **"AFTER UPDATE"** triggers must be created;

---
###### Table Creation:
---
```sql
     CREATE TABLE categories
        ( id 					INT NOT NULL  PRIMARY KEY AUTO_INCREMENT
         , category_level   	INT(1) NOT NULL
         , parentid 			INT     NULL
         , foreign key FK_parentid (parentid) 
           REFERENCES categories (id)
     ) ENGINE=InnoDB
```

---
###### Trigger: Before INSERT:
---
```sql
 CREATE TRIGGER checkSubCategoryLevel BEFORE INSERT ON category
 FOR EACH ROW BEGIN
    IF (NEW.category_level < 0 OR NEW.category_level > 5) THEN
         INSERT INTO category(category_level) VALUES(0);
    END IF;
 END
```
---
###### Trigger: After UPDATE:
---
	
```sql
 CREATE TRIGGER checkSubCategoryLevelUpdate AFTER UPDATE ON categories
 FOR EACH ROW BEGIN
    IF (NEW.category_level < 0 OR NEW.category_level > 5) THEN
         INSERT INTO categories(category_level) VALUES(OLD.category_level);
    END IF;
 END
``` 

	- Data insertion (see Logs section 5.1)
		- To achieve greater simplicity, I assumed the ID structure given in the example:
		ID:	134526		
		1 - TOP LEVEL 0 ID
		3 - Sub level 1 ID
		4 - Sub level 2 ID
		5 - Sub level 3 ID
		2 - Sub level 4 ID
		6 - Sub level 5 ID



```sql
CREATE PROCEDURE `populateCategories`()

BEGIN
        
        DECLARE numCat INT DEFAULT 5;
        DECLARE it, currentit, currentit2, currentit3, currentit4, currentit5 INT DEFAULT 1;
        
        START TRANSACTION;
		INSERT INTO log_db (id, times0, times1, times2, times3, times4, times5) VALUES (NULL, 0, 0, 0, 0, 0, 0);           
        
        -- CREATE LEVEL 0
        WHILE it <= numCat DO			
			INSERT INTO categories (id, category_level, parentid) VALUES(it, 0, null);           
            UPDATE log_db SET times0  = it;
            -- CREATE LEVEL 1
            WHILE currentit <= numCat DO
              INSERT INTO categories (id, category_level, parentid) VALUES(concat(it, currentit), 1, it);             
              UPDATE log_db SET times1  = currentit * it;
              -- CREATE LEVEL 2
              WHILE currentit2 <= numCat DO
				  INSERT INTO categories (id, category_level, parentid) VALUES(concat(it, concat(currentit, currentit2)), 2, concat(it, currentit));				  
				  UPDATE log_db SET times2  = currentit2 * currentit * it;                  
                   -- CREATE LEVEL 3
                  WHILE currentit3 <= numCat DO
						INSERT INTO categories (id, category_level, parentid) VALUES(concat(it, concat(currentit, concat(currentit2, currentit3))), 3, concat(it, concat(currentit, currentit2)));					
						UPDATE log_db SET times3  = currentit3 *currentit2 * currentit * it ;                       
						-- CREATE LEVEL 4
                        WHILE currentit4 <= numCat DO
							INSERT INTO categories (id, category_level, parentid) VALUES(concat(it, concat(currentit, concat(currentit2, concat(currentit3, currentit4)))), 4, concat(it, concat(currentit, concat(currentit2, currentit3))));					
							UPDATE log_db SET times4  = currentit4 * currentit3 * currentit2 * currentit * it;
                             -- CREATE LEVEL 5
                            WHILE currentit5 <= numCat DO
								INSERT INTO categories (id, category_level, parentid) VALUES(concat(it, concat(currentit, concat(currentit2, concat(currentit3, concat(currentit4, currentit5))))), 5, concat(it, concat(currentit, concat(currentit2, concat(currentit3, currentit4)))));					
								UPDATE log_db SET times5  = currentit5 * currentit4 * currentit3 * currentit2 * currentit * it;      
                                SET currentit5 = currentit5 + 1;
                                END WHILE;                             
                            SET currentit4 = currentit4 + 1;
							SET currentit5 = 1;  
                         END WHILE;   							
						SET currentit3 = currentit3 + 1;
						SET currentit4 = 1;
				  END WHILE;
                  SET currentit2 = currentit2 + 1;
				  SET currentit3 = 1;
			  END WHILE;
			  SET currentit = currentit + 1;
			  SET currentit2 = 1;
            END WHILE;
            SET currentit = 1;
            SET it = it + 1;
		END WHILE;
        COMMIT;
    END
```

#### 5.2.1
<p> Since my design included a <b>"category_level"</b> field, I only need to execute a <b>"SELECT"</b> command with a <b>"WHERE"</b> clause indicating the <b>"category_level"</b> equals 0. </p>

---
```sql
 SELECT * FROM categories
	WHERE category_level = 0
```

#### 5.2.2
<p> Assuming the data architecture described in the section 5.1, the <b>"LEFT(parentid, 1)"</b> command needs to be executed in order to retrieve the ID of the parent top level id.</p> 
---
* Select all Sub Categories level 3 from Parent TOP level 2 = 2:

```sql
 SELECT * FROM categories
	WHERE category_level = 3 AND (LEFT(parentid, 1) = 2)
```
 
#### 5.2.3
<p> Since the operator <b>"LIKE"</b> cannot be used, I decided to use the operators <b>"RIGHT"</b> and <b>"LEFT"</b> in conjuction to obtain the desired result. </p>
---
* Select all Sub Categories level 4 from Parent TOP level 2 = 5:

```sql
 SELECT * FROM categories
	WHERE category_level = 4 AND  RIGHT(LEFT(parentid, 2), 1) = 5	
```	

#### 5.2.4
<p> The use of the command <b>"COUNT(*)"</b> in conjunction with the clause <b>"WHERE"</b> specifying the sub category 4 produces the desired result. </p>
---
```sql
 SELECT COUNT(*) FROM categories
	WHERE category_level = 3	
```