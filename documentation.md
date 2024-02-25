**SBD PROJECT 23/24**

> **Deniz Tolga Tezcan  
> s26772  
> Group: 24c**

-   This project’s aim is to create a database for a Green Grocery Store
    with procedures and triggers.

-   The database consists of Product, Category, Customer, Supplier,
    Supply_Product, Purchase, Basket, Deleted_Products, Price_Changes
    entities.

***PL/SQL***

**BEFORE INSERT (ROW TRIGGER)**

-   Creates primary key automatically if it was null during insertion
    operation.

CREATE OR REPLACE TRIGGER auto_increase_id

BEFORE INSERT ON PRODUCT

FOR EACH ROW

DECLARE

v_id INTEGER;

BEGIN

SELECT NVL(MAX(PR_ID),0) + 1

INTO v_id FROM PRODUCT;

IF :NEW.PR_ID IS NULL THEN

:NEW.PR_ID := v_id;

END IF;

END;

**AFTER INSERT OR UPDATE (ROW TRIGGER)**

-   Updates price and expiration date automatically on PRODUCT table
    after inserting to SUPPLY_PRODUCT table or updating the
    SUPPLY_PRODUCT table.

CREATE OR REPLACE TRIGGER update_price_and_date_trigger

AFTER INSERT OR UPDATE OF COST,KG,DELIVERY_DATE

ON SUPPLY_PRODUCT

FOR EACH ROW

DECLARE

v_price PRODUCT.PRICE%TYPE;

v_date DATE;

BEGIN

v_price := (:NEW.COST / :NEW.KG) + 2;

v_date := ADD_MONTHS(:NEW.DELIVERY_DATE,2);

UPDATE PRODUCT SET PRICE = v_price WHERE PR_ID = :NEW.PR_ID;

UPDATE PRODUCT SET EXP_DATE = v_date WHERE PR_ID = :NEW.PR_ID;

END;

**AFTER UPDATE (ROW TRIGGER)**

-   After updating a price of a product, triggers insert this changes to
    PRICE_CHANGES table also automatically assign primary key for
    PRICE_CHANGES table.

CREATE OR REPLACE TRIGGER UPDATE_PRODUCT_PRICE

AFTER UPDATE OF PRICE

ON PRODUCT

FOR EACH ROW

DECLARE

v_old_price PRODUCT.PRICE%TYPE;

v_new_price PRODUCT.PRICE%TYPE;

v_id INTEGER;

BEGIN

v_old_price := :OLD.PRICE;

v_new_price := :NEW.PRICE;

SELECT NVL(MAX(ID),0) + 1 INTO v_id

FROM PRICE_CHANGES;

IF :OLD.PRICE \<\> :NEW.PRICE THEN

INSERT INTO PRICE_CHANGES
VALUES(v_id,v_old_price,v_new_price,:OLD.PR_ID,sysdate);

END IF;

END;

**BEFORE UPDATE (CURSOR)**

-   It gives information about the current prices of the all products
    before updating the SUPPLY_PRODUCT table.

-   Assumed “SET SERVEROUTPUT ON” executed.

CREATE OR REPLACE TRIGGER before_update_trigger

BEFORE UPDATE OF COST,KG,DELIVERY_DATE

ON SUPPLY_PRODUCT

FOR EACH ROW

DECLARE

CURSOR my_cursor IS

SELECT PR_NAME, PRICE

FROM PRODUCT;

v_record my_cursor%ROWTYPE;

BEGIN

FOR v_record IN my_cursor LOOP

DBMS_OUTPUT.PUT_LINE(v_record.PR_NAME \|\| ': ' \|\| v_record.PRICE);

END LOOP;

END;

**AFTER INSERT (TABLE TRIGGER)**

-   After inserting a record to PRODUCT table, it gives information
    about the most expensive price in the green grocery store.

CREATE or REPLACE TRIGGER after_insert_product_expensive

AFTER INSERT

ON PRODUCT

DECLARE

v_price PRODUCT.PRICE%TYPE;

BEGIN

SELECT MAX(PRICE) INTO v_price

FROM PRODUCT;

DBMS_OUTPUT.PUT_LINE('The most expensive price in the store is ' \|\|
v_price);

END;

**BEFORE DELETE(ROW TRIGGER)**

-   The trigger deletes all the related foreign keys in other tables
    before deleting the record from PRODUCT table.

CREATE OR REPLACE TRIGGER before_delete

BEFORE DELETE

ON PRODUCT

FOR EACH ROW

DECLARE

v_count INTEGER;

BEGIN

SELECT COUNT(\*) INTO v_count FROM SUPPLY_PRODUCT WHERE PR_ID =
:OLD.PR_ID;

IF v_count \> 0 THEN

DELETE FROM SUPPLY_PRODUCT

WHERE PR_ID = :OLD.PR_ID;

DBMS_OUTPUT.PUT_LINE(v_count \|\| ' foreign key(s) deleted from
SUPPLY_PRODUCT');

END IF;

SELECT COUNT(\*) INTO v_count FROM BASKET WHERE PR_ID = :OLD.PR_ID;

IF v_count \> 0 THEN

DELETE FROM BASKET

WHERE PR_ID = :OLD.PR_ID;

DBMS_OUTPUT.PUT_LINE(v_count \|\| ' foreign key(s) deleted from
BASKET');

END IF;

SELECT COUNT(\*) INTO v_count FROM PRICE_CHANGES WHERE PR_ID =
:OLD.PR_ID;

IF v_count \> 0 THEN

DELETE FROM PRICE_CHANGES

WHERE PR_ID = :OLD.PR_ID;

DBMS_OUTPUT.PUT_LINE(v_count \|\| ' foreign key(s) deleted from
PRICE_CHANGES');

END IF;

END;

**AFTER DELETE(ROW TRIGGER)**

-   The trigger insert a record which is deleted to DELETED_PRODUCTS
    table and prints information about which product is deleted.

CREATE OR REPLACE TRIGGER after_delete

AFTER DELETE

ON PRODUCT

FOR EACH ROW

DECLARE

v_id INTEGER;

BEGIN

SELECT NVL(MAX(ID),0) + 1 INTO v_id FROM DELETED_PRODUCTS;

INSERT INTO DELETED_PRODUCTS VALUES (v_id,:OLD.PR_NAME,SYSDATE);

DBMS_OUTPUT.PUT_LINE(:OLD.PR_NAME \|\| ' is deleted.');

END;

**PROCEDURE-1 (CURSOR)**

-   Applies discount for the products which are about to be expired in
    15 days.

CREATE OR REPLACE PROCEDURE discount

AS

CURSOR c_product IS SELECT \* FROM PRODUCT;

v_discount NUMBER := 0.50;

v_old_price NUMBER;

BEGIN

FOR r_product IN c_product LOOP

IF (r_product.EXP_DATE - SYSDATE) \<= 15 THEN

v_old_price := r_product.PRICE;

r_product.PRICE := r_product.PRICE \* v_discount;

UPDATE PRODUCT

SET PRICE = r_product.PRICE

WHERE PR_ID = r_product.PR_ID;

DBMS_OUTPUT.PUT_LINE(r_product.PR_NAME \|\| ': ' \|\| v_old_price \|\| '
-\> ' \|\| r_product.PRICE);

END IF;

END LOOP;

END;

**PROCEDURE-2 (EXCEPTION)**

-   This procedure checks if there is enough amount of product to be
    added. If there is not, it raises exception.

CREAET OR REPLACE PROCEDURE add_basket(v_b_id INTEGER,v_p_id
INTEGER,v_kg NUMBER DEFAULT 1, v_purc_id INTEGER)

AS

v_available_product NUMBER;

not_enough_product EXCEPTION;

BEGIN

v_available_product := measure_available_products(v_p_id);

IF v_available_product - v_kg \< 0 THEN

RAISE not_enough_product;

ELSE

INSERT INTO BASKET VALUES(v_b_id, v_kg, v_p_id, v_purc_id);

END IF;

EXCEPTION

WHEN not_enough_product THEN

DBMS_OUTPUT.PUT_LINE('Not enough product');

END;

**PROCEDURE-3 (CURSOR)**

-   It shows the products which are imported. (Assumed the green grocery
    store is located in Poland).

CREATE OR REPLACE PROCEDURE show_imports

AS

CURSOR c_products IS

SELECT S_COUNTRY,PR_NAME

FROM SUPPLY_PRODUCT,PRODUCT,SUPPLIER

WHERE SUPPLY_PRODUCT.PR_ID = PRODUCT.PR_ID

AND SUPPLY_PRODUCT.S_ID = SUPPLIER.S_ID

AND S_COUNTRY \<\> 'POLAND';

BEGIN

FOR c_row IN c_products LOOP

DBMS_OUTPUT.PUT_LINE(c_row.PR_NAME \|\| ' -\> ' \|\| c_row.S_COUNTRY);

END LOOP;

END;

**ADDITIONAL FUNCTION**

-   It computes the available kg of the given product id.

create or replace FUNCTION measure_available_products(v_pr_id INTEGER)

RETURN SUPPLY_PRODUCT.KG%TYPE

IS

v_result SUPPLY_PRODUCT.KG%TYPE := 0;

v_total SUPPLY_PRODUCT.KG%TYPE := 0;

v_sold SUPPLY_PRODUCT.KG%TYPE := 0;

v_date DATE;

v_count INTEGER;

BEGIN

SELECT COUNT(\*) INTO v_count FROM SUPPLY_PRODUCT WHERE PR_ID = v_pr_id;

IF v_count \> 0 THEN

SELECT MAX(DELIVERY_DATE) INTO v_date

FROM SUPPLY_PRODUCT

WHERE PR_ID = v_pr_id;

SELECT SUM(KG) INTO v_total

FROM SUPPLY_PRODUCT

WHERE PR_ID = v_pr_id AND DELIVERY_DATE = v_date;

ELSE

RETURN v_result;

END IF;

SELECT COUNT(\*) INTO v_count FROM BASKET WHERE PR_ID = v_pr_id;

IF v_count \> 0 THEN

SELECT SUM(KG) INTO v_sold

FROM BASKET,PURCHASE

WHERE BASKET.PURCHASE_ID = PURCHASE.PURCHASE_ID

AND PR_ID = v_pr_id

AND v_date \<= P_DATE;

IF v_sold IS NULL THEN

RETURN v_total;

ELSE

v_result := v_total - v_sold;

RETURN v_result;

END IF;

ELSE

RETURN v_total;

END IF;

END;

***T – SQL***

**1-**

CREATE TRIGGER auto_increase_id

ON PRODUCT

INSTEAD OF INSERT

AS

BEGIN

DECLARE @v_id INT;

SELECT @v_id = ISNULL(MAX(PR_ID),0) + 1

FROM PRODUCT;

INSERT INTO PRODUCT(PR_ID,PR_NAME,CAT_ID,PRICE,EXP_DATE)

SELECT

ISNULL(PR_ID,@v_id),PR_NAME,CAT_ID,PRICE,EXP_DATE

FROM INSERTED;

END;

**2-**

CREATE TRIGGER update_price_and_date_trigger

ON SUPPLY_PRODUCT

FOR INSERT,UPDATE

AS

BEGIN

DECLARE @v_price NUMERIC(7,2),@v_date DATE;

SELECT @v_price = COST / KG + 2,

@v_date = DATEADD(MONTH, 2, DELIVERY_DATE)

FROM INSERTED;

IF UPDATE(COST) OR UPDATE(KG) OR UPDATE(DELIVERY_DATE)

BEGIN

UPDATE P

SET P.PRICE = @v_price,

P.EXP_DATE = @v_date

FROM PRODUCT P,INSERTED I

WHERE P.PR_ID = I.PR_ID;

END

END;

**3-**

CREATE TRIGGER update_product_price

ON PRODUCT

FOR UPDATE

AS

BEGIN

DECLARE @v_old_price NUMERIC(7,2), @v_new_price NUMERIC(7,2),@v_id INT,
@v_pr_id INT;

IF UPDATE(PRICE)

BEGIN

DECLARE price_cursor CURSOR FOR

SELECT d.PR_ID, d.PRICE,i.PRICE

FROM DELETED D,INSERTED I

WHERE I.PR_ID = D.PR_ID

AND D.PRICE \<\> I.PRICE

AND D.PRICE IS NOT NULL;

OPEN price_cursor;

FETCH NEXT FROM price_cursor INTO @v_pr_id,@v_old_price,@v_new_price;

WHILE @@FETCH_STATUS = 0

BEGIN

SELECT @v_id = ISNULL(MAX(ID),0) + 1 FROM PRICE_CHANGES;

INSERT INTO PRICE_CHANGES VALUES
(@v_id,@v_old_price,@v_new_price,@v_pr_id,GETDATE());

FETCH NEXT FROM price_cursor INTO @v_pr_id,@v_old_price,@v_new_price;

END;

CLOSE price_cursor;

DEALLOCATE price_cursor;

END

END;

**4-**

CREATE TRIGGER price_trigger_cursor

ON PRICE_CHANGES

FOR INSERT

AS

BEGIN

DECLARE my_cursor CURSOR FOR SELECT PR_NAME, PRICE FROM PRODUCT;

DECLARE @v_pr_name VARCHAR(20),@v_price NUMERIC(7,2);

OPEN my_cursor;

FETCH NEXT FROM my_cursor INTO @v_pr_name,@v_price;

WHILE @@FETCH_STATUS = 0

BEGIN

PRINT @v_pr_name + ': ' + CAST(@v_price AS VARCHAR);

FETCH NEXT FROM my_cursor INTO @v_pr_name,@v_price;

END

CLOSE my_cursor;

DEALLOCATE my_cursor;

END;

**5-**

CREATE TRIGGER expensive_product

ON PRODUCT

FOR INSERT

AS

BEGIN

DECLARE @v_price NUMERIC(7,2);

SELECT @v_price = MAX(PRICE) FROM PRODUCT;

PRINT 'The most expensive product in the store is ' + CAST(@v_price AS
VARCHAR);

END;

**6-**

ALTER TRIGGER delete_fk_trigger

ON PRODUCT

INSTEAD OF DELETE

AS

BEGIN

DECLARE @v_count INT, @v_id INT, @v_name VARCHAR(20);

SELECT @v_id = PR_ID FROM DELETED;

IF NOT EXISTS (SELECT 1 FROM PRODUCT WHERE PR_ID = @v_id)

BEGIN

ROLLBACK;

RAISERROR('ID does not exist in PRODUCT table',1,2);

END

DECLARE supply_product_cursor CURSOR LOCAL FOR

SELECT COUNT(\*) FROM SUPPLY_PRODUCT

WHERE PR_ID = @v_id;

OPEN supply_product_cursor;

FETCH NEXT FROM supply_product_cursor INTO @v_count;

CLOSE supply_product_cursor;

IF @v_count \> 0

BEGIN

DELETE FROM SUPPLY_PRODUCT

WHERE PR_ID = @v_id;

PRINT CAST(@v_count AS VARCHAR) + ' foreign key(s) deleted from
SUPPLY_PRODUCT.';

END

DECLARE basket_cursor CURSOR LOCAL FOR

SELECT COUNT(\*)

FROM BASKET

WHERE PR_ID = @v_id;

OPEN basket_cursor;

FETCH NEXT FROM basket_cursor INTO @v_count;

CLOSE basket_cursor;

IF @v_count \> 0

BEGIN

DELETE FROM BASKET

WHERE PR_ID = @v_id;

PRINT CAST(@v_count AS VARCHAR) + ' foreign key(s) deleted from
BASKET.';

END

DECLARE price_changes_cursor CURSOR LOCAL FOR

SELECT COUNT(\*)

FROM PRICE_CHANGES

WHERE PR_ID = @v_id;

OPEN price_changes_cursor;

FETCH NEXT FROM price_changes_cursor INTO @v_count;

CLOSE price_changes_cursor;

IF @v_count \> 0

BEGIN

DELETE FROM PRICE_CHANGES

WHERE PR_ID = @v_id;

PRINT CAST(@v_count AS VARCHAR) + ' foreign key(s) deleted from
PRICE_CHANGES';

END

DELETE FROM PRODUCT

WHERE PR_ID = @v_id;

SELECT @v_name = PR_NAME FROM DELETED;

PRINT @v_name + ' deleted';

DECLARE @v_d_id INT;

SELECT @v_d_id = ISNULL(MAX(ID),0) + 1 FROM DELETED_PRODUCTS;

INSERT INTO DELETED_PRODUCTS VALUES(@v_id,@v_name,GETDATE());

END;

**PROCEDURE-1 (RESULT SET)**

CREATE PROCEDURE show_products_by_country

@v_country VARCHAR(20)

AS

BEGIN

SELECT PR_NAME

FROM PRODUCT,SUPPLIER,SUPPLY_PRODUCT

WHERE PRODUCT.PR_ID = SUPPLY_PRODUCT.PR_ID

AND SUPPLIER.S_ID = SUPPLY_PRODUCT.S_ID

AND S_COUNTRY = @v_country;

END;

**PROCEDURE-2 (RETURN)**

CREATE PROCEDURE show_total_number_of_fruits_or_vegetables_by_cat_id

@cat_id INT

AS

BEGIN

DECLARE @result INT;

SELECT @result = COUNT(\*) FROM PRODUCT WHERE CAT_ID = @cat_id;

RETURN @result;

END;

**PROCEDURE-3 (OUTPUT)**

CREATE PROCEDURE show_how_many_times_bought

@v_count INTEGER OUTPUT,

@v_id INTEGER

AS

BEGIN

SELECT @v_count = COUNT(\*) FROM BASKET WHERE PR_ID = @v_id;

END;
