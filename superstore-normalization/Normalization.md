# Normalization

# This is Superstore Sales Dataset. In this notebook we're gonna Normalize this dataset.

```sql
select * from orders;
```

*First of all w'll create a date dimension table to empower the dataset for time intellengence.*

```sql
-- Creation of Dim_date
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY AUTO_INCREMENT,
    full_date DATE NOT NULL UNIQUE,
    year INT NOT NULL,
    quarter INT NOT NULL,
    quarter_name VARCHAR(5) NOT NULL,
    month INT NOT NULL,
    month_name VARCHAR(15) NOT NULL,
    week_of_year INT NOT NULL,
    day_of_week INT NOT NULL,
    day_name VARCHAR(15) NOT NULL,
    is_weekend BOOLEAN NOT NULL
) ENGINE=InnoDB;
```

*Now we'll create a stored procedure to insert the date range into the date dimesion table for the specified date range.*

```sql
DELIMITER //

CREATE PROCEDURE sp_populate_dim_date(
    IN p_start_date DATE,
    IN p_end_date DATE
)
BEGIN
    -- Dynamically adjust recursion limit based on the requested date range
    SET SESSION cte_max_recursion_depth = DATEDIFF(p_end_date, p_start_date) + 10;

    INSERT IGNORE INTO dim_date (
        full_date,
        year,
        quarter,
        quarter_name,
        month,
        month_name,
        week_of_year,
        day_of_week,
        day_name,
        is_weekend
    )
    WITH RECURSIVE date_series AS (
        SELECT p_start_date AS dt
        UNION ALL
        SELECT dt + INTERVAL 1 DAY
        FROM date_series
        WHERE dt < p_end_date
    )
    SELECT 
        dt AS full_date,
        YEAR(dt) AS year,
        QUARTER(dt) AS quarter,
        CONCAT('Q', QUARTER(dt)) AS quarter_name,
        MONTH(dt) AS month,
        MONTHNAME(dt) AS month_name,
        WEEKOFYEAR(dt) AS week_of_year,
        WEEKDAY(dt) + 1 AS day_of_week,
        DAYNAME(dt) AS day_name,
        IF(WEEKDAY(dt) >= 5, 1, 0) AS is_weekend -- Saturday (5), Sunday (6)
    FROM date_series;
END //

DELIMITER ;
```

*Now let's find the date range that is currently required for our superstore dataset*

```sql
With date_range as(
Select Min(`order date`) as start_date, Max(`order date`) as end_date from orders
UNION
Select Min(`ship date`) as start_date, Max(`ship date`) as end_date from orders)
select Min(start_date) as start_date, max(end_date) as end_date
from date_range;
```

*Above we obtained the date range, so currently we will insert the date range 01-01-2014 to 31-12-2018 into the dim_date table, using the stored procedure.*

```sql
CALL sp_populate_dim_date('2014-01-01', '2018-12-31');
```

```sql
select * from dim_date;
```

*NOw, we'll create the Dim_address table, that will be referenced by Dim_customer*

```sql
Create table dim_address as(
with clean_data as (
    Select Trim(Country) as Country, Trim(Region) as Region, Trim(state) as State, Trim(city) as City, Lpad(`postal code`, 5, '0') as Postal_code
    from orders
)

select
(ROW_NUMBER() over(order by Country, Region, State, City, postal_code) + 100) as Address_id,
Country, Region, state, city, Concat(State, '- ', City) as State_city, postal_code
from clean_data group by Country, Region, state, city, postal_code);
```

```sql
desc dim_address;
```

```sql
Alter table dim_address
modify Address_id int primary KEY,
modify country varchar(50) not null default 'United States',
modify Region Varchar(50) not null,
modify State Varchar(50) not null,
modify City Varchar(50) not null,
modify State_city Varchar(200) not null,
modify Postal_code Varchar(20) not null,

ADD CONSTRAINT UQ_Dim_Address_Location 
        UNIQUE (Country, Region, State, City, Postal_code);
```

```sql
Select * from dim_address;
```

*Now let's work on Customer Dimension table*

Because one customer can have multiple address as they can make orders from different places, so we will link the address_id in direct fact table rather that the customer dimension table.

```sql
Create table dim_customer as(
Select `Customer ID` as Customer_id,
`Customer Name` as Customer_name,
Segment
from orders
group by
 `Customer ID`,
`Customer Name`,
Segment);
```

```sql
Desc dim_customer;
```

```sql
Alter table dim_customer
Modify Customer_id varchar(50) Primary Key,
Modify Customer_name varchar(200) not null,
Modify Segment varchar(50) not null;
```

```sql
select * from dim_customer;
```

*Next we will build category dimension table which will refrenced by the Product table*

```sql
CREATE table dim_product_category(
    Category_id int primary key AUTO_INCREMENT,
    Category varchar(100) not null,
    Sub_category varchar(100) not null
);
```

```sql
Insert into dim_product_category (Category_id, Category, Sub_category)
(    Select
        Row_number() over(order by Category, `Sub-Category`) as Category_id,
        Category,
        `Sub-Category` as Sub_category
    from orders
    group by
            Category,
            `Sub-Category`
);
```

```sql
select * from dim_product_category;
```

*Now, we will create a product table that will reference the category table*

```sql
Create Table dim_product(
    Product_sk int primary Key AUTO_INCREMENT,
    Product_id varchar(50) not null,
    Product_name varchar(500) not null,
    Category_id int not null,

    Constraint fk_product_category
        FOREIGN KEY (Category_id)
        References dim_product_category(Category_id)
);
```

```sql
Insert Into dim_product (Product_sk, Product_id, Product_name, Category_id)
(Select
ROW_NUMBER() over(order by o.`Product Name`) as Product_sk,
o.`Product ID` as Product_id,
o.`Product Name` as Product_name, c.Category_id
from orders o left join dim_product_category c
on o.Category = c.Category COLLATE utf8mb4_0900_ai_ci and
o.`Sub-Category` = c.Sub_category COLLATE utf8mb4_0900_ai_ci
Group by o.`Product ID`, o.`Product Name`, c.category_id);
```

```sql
Select * from dim_product;
```

*Now at very last, let's create The main fact orders table*

```sql
Create table fact_orders (
    Fact_order_id int Primary Key AUTO_INCREMENT,
    Order_id varchar(50) not null,
    Order_line_num int not null,
    Order_date_id int not null,
    Ship_date_id int not null,
    Ship_mode varchar(50) not null,
    Customer_id varchar(50) not null,
    Address_id int not null,
    Product_sk int not null,
    Sales Decimal(10, 4) not null check(Sales >= 0.00),
    Quantity int not null check(Quantity >= 0),
    Discount Decimal(4, 2) not null check(Discount >= 0),
    Profit Decimal(10, 4) not null,
    Constraint fk_fact_order_date Foreign Key (Order_date_id) References dim_date(date_key),
    Constraint fk_fact_ship_date Foreign Key (Ship_date_id) References dim_date(date_key),
    Constraint fk_fact_customer Foreign Key (Customer_id) References dim_customer(Customer_id),
    Constraint fk_fact_address Foreign Key (Address_id) References dim_address(Address_id),
    Constraint fk_fact_product Foreign Key (Product_sk) References dim_product(Product_sk)

);
```

```sql
Insert Into fact_orders (
    Order_id,
    Order_line_num,
    Order_date_id,
    Ship_date_id,
    Ship_mode,
    Customer_id,
    Address_id,
    Product_sk,
    Sales,
    Quantity,
    Discount,
    Profit
)

(Select 
o.`Order ID` as Order_id,
ROW_NUMBER() over(partition by o.`Order ID` Order by o.`Order ID`) as Order_line_num,
d.date_key as Order_date_id,
d2.date_key as Ship_date_id,
o.`Ship Mode` as Ship_mode,
o.`Customer ID` as Customer_id,
a.Address_id,
p.Product_sk,
o.Sales,
o.Quantity,
o.Discount,
o.Profit

from orders o 

left join dim_date d
on o.`Order Date` = d.full_date

left join dim_date d2
on o.`Ship Date` = d2.full_date

left join dim_address a
on o.Country = a.Country COLLATE utf8mb4_0900_ai_ci
and o.Region = a.Region COLLATE utf8mb4_0900_ai_ci
and o.State = a.State COLLATE utf8mb4_0900_ai_ci
and o.City = a.City COLLATE utf8mb4_0900_ai_ci
and Lpad(Trim(o.`Postal Code`), 5, '0') = a.Postal_code COLLATE utf8mb4_0900_ai_ci

left join dim_product p
on o.`Product ID` = p.Product_id COLLATE utf8mb4_0900_ai_ci
and o.`Product Name` = p.Product_name COLLATE utf8mb4_0900_ai_ci);
```

```sql
Select

o.Fact_order_id,
o.Order_id,
o.Order_line_num,
d.full_date as Order_date,
d1.full_date as Ship_date,
o.Ship_mode,
o.Customer_id,
c.Customer_name,
c.Segment,
a.Country,
a.region,
a.State,
a.City,
a.State_city,
a.Postal_code,
p.Product_id,
p.Product_name,
pc.Category,
pc.Sub_category,
o.Sales,
o.Quantity,
o.Discount,
o.Profit

from fact_orders o
Left join dim_date d on o.Order_date_id = d.date_key
Left join dim_date d1 on o.Order_date_id = d1.date_key
Left join dim_customer c on o.Customer_id = c.Customer_id
Left join dim_address a on o.Address_id = a.Address_id
Left Join dim_product p on o.Product_sk = p.Product_sk
Left join dim_product_category pc on p.Category_id = pc.Category_id;
```
