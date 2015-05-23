#3. Working with the database
##3.1 Database design
###3.1.1 Introduction

A study about database design and data structures deserves many books on itself. Therefore, we will try to cover the most common good design practices that can serve as a starting point when designing your application's requirements data structures.
Database desing is a very important step in software development lifecycle because any design decision can dramatically affect the project's performance and maintainance cost.

At this point we are going to use an RDBMS (Relational database management system). We will talk about other storages engines like noSql later.

When designing a database schema, there are two major questions you should ask yourself:

1.  How I am going to query the data?  
Consider the follwing simple table
````sql
`user_account` (
  `id` INT(10),
  `login` VARCHAR(45) NULL DEFAULT NULL,
  `password` VARCHAR(100) NULL DEFAULT NULL,
  `email` VARCHAR(45) NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
ENGINE = InnoDB;
````
Knowing that I will execute queries like `SELECT * from user_account WHERE email like "search_pattern%"` I may think about adding an **index** on user_account.email column to speed up this select query. However, if I know that most of the queries will be like `SELECT * from user_account WHERE email like "%search_pattern%"`, adding an index will have no effect except increasing the table index size.

2.  How the data will be distributed?  
How many records this table will contain? How many records will have a NULL value in this column? What cardinality this column is going to have? Accurately predicting answers to those questions can be a tricky task. I've seen tables designed to hold *no more than 10 records* that ended up with thousands of entries.

When comparing design variants, there are three metrics that interests the application developer: read speed, update speed, and data storage size. Usually the last one is the least of your concerns, although it can be very important in some situations. You often need to make a balance between read and update speed depending on your application's needs.

An example comparaison:  
Benchmark with 5000000 records

<table>
    <tr>
        <th>Variant</th>
        <th>Avg SELECT speed</th>
        <th>Avg UPDATE speed</th>
        <th>Index size</th>
    </tr>
    <tr>
        <td>With index</td>
        <td>50 ms</td>
        <td>800 ms</td>
        <td>250 MB</td>
    </tr>
    <tr>
        <td>Without index</td>
        <td>400 ms</td>
        <td>200 ms</td>
        <td>0</td>
    </tr>
</table>

There is absolutely no generic pattern on making a decision in this case. It solely depends on your specific application's needs.



###3.1.2 Data schema

We are still adopting the target bullet process. That means our first database design is not final and we will eventually update it as needed while going through implementing new features.

For this tutorial, we are going tu use MySQL database server. That said, the SQL syntax will be MySQL.

- Create the schema

````sql
CREATE SCHEMA IF NOT EXISTS `symfony` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
````
- Category table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`category` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `label` VARCHAR(45) NULL,
  `parent_category_id` INT UNSIGNED NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  INDEX `fk_category_category_idx` (`parent_category_id` ASC),
  CONSTRAINT `fk_category_category`
    FOREIGN KEY (`parent_category_id`)
    REFERENCES `symfony`.`category` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB;
````

>Whenever you can, take advantage of the dabase server new ID generation strategy for your primary keys (AUTO_INCREMENT for MySQL) You may be tempted to develop your own generation strategy (example: pattern based IDs like "day_date_sequential_number"), don't. You can have such a column in your table but keep it separate from table **id** key.  

<br/>

>When using auto generated keys, make them **UNSIGNED**. The values will be all positive (by default) so there is no point of deviding your values range by 2. For example the INT type for MySQL has the follwing ranges:
<table>
  <tr>
    <th></th>
    <th>Minumum value</th>
    <th>Maximum value</th>
  </tr>
  <tr>
    <td>Signed</td>
    <td>-2147483648</td>
    <td>2147483647</td>
  </tr>
  <tr>
    <td>Unsigned</td>
    <td>0</td>
    <td>4294967295</td>
  </tr>
</table>

- product table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`product` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `code` VARCHAR(45) NULL DEFAULT NULL,
  `title` VARCHAR(200) NULL DEFAULT NULL,
  `description` TEXT NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  INDEX `code_index` (`code` ASC),
  INDEX `title_index` (`title` ASC))
ENGINE = InnoDB;
````


- a product belongs to many categories

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`category_has_product` (
  `category_id` INT(10) UNSIGNED NOT NULL,
  `product_id` INT(10) UNSIGNED NOT NULL,
  PRIMARY KEY (`category_id`, `product_id`),
  INDEX `fk_category_has_product_product_idx` (`product_id` ASC),
  INDEX `fk_category_has_product_category_idx` (`category_id` ASC),
  CONSTRAINT `fk_category_has_product_category`
    FOREIGN KEY (`category_id`)
    REFERENCES `symfony`.`category` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_category_has_product_product`
    FOREIGN KEY (`product_id`)
    REFERENCES `symfony`.`product` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8
COLLATE = utf8_general_ci;
````


- warehouse table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`warehouse` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(45) NULL DEFAULT NULL,
  `address` TEXT NULL DEFAULT NULL,
  PRIMARY KEY (`id`))
ENGINE = InnoDB;
````


- a product can be present in many warehouses but with different quantities

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`warehouse_has_product` (
  `warehouse_id` INT(10) UNSIGNED NOT NULL,
  `product_id` INT(10) UNSIGNED NOT NULL,
  `quantity` INT(11) NULL DEFAULT NULL,
  PRIMARY KEY (`warehouse_id`, `product_id`),
  INDEX `fk_warehouse_has_product_product_idx` (`product_id` ASC),
  INDEX `fk_warehouse_has_product_warehouse_idx` (`warehouse_id` ASC),
  CONSTRAINT `fk_warehouse_has_product_warehouse`
    FOREIGN KEY (`warehouse_id`)
    REFERENCES `symfony`.`warehouse` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_warehouse_has_product_product1`
    FOREIGN KEY (`product_id`)
    REFERENCES `symfony`.`product` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB;
````

- a product can be put in sale for a specific period with a given sale price

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`product_sale` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `active` TINYINT(4) NULL DEFAULT NULL,
  `start_date` DATETIME NULL DEFAULT NULL,
  `end_date` DATETIME NULL DEFAULT NULL,
  `price` INT(11) NULL DEFAULT NULL,
  `product_id` INT(10) UNSIGNED NOT NULL,
  PRIMARY KEY (`id`),
  INDEX `fk_product_sale_product_idx` (`product_id` ASC),
  INDEX `start_date_index` (`start_date` ASC),
  INDEX `end_date_index` (`end_date` ASC),
  CONSTRAINT `fk_product_sale_product`
    FOREIGN KEY (`product_id`)
    REFERENCES `symfony`.`product` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB;
````

> When working with MySQL prefer DATETIME above TMESTAMP. TMESTAMP has nothing more than DATETIME, however it has some drowbacks like having a maximum value of 2038-01-09 03:14:07 UTC


- vendor table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`vendor` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(45) NULL DEFAULT NULL,
  PRIMARY KEY (`id`))
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8
COLLATE = utf8_general_ci;
````

- product_acquisition table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`product_acquisition` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `date` DATETIME NULL DEFAULT NULL,
  `quantity` INT(11) NULL DEFAULT NULL,
  `price` INT(11) NULL DEFAULT NULL,
  `vendor_id` INT(10) UNSIGNED NOT NULL,
  `product_id` INT(10) UNSIGNED NOT NULL,
  PRIMARY KEY (`id`),
  INDEX `fk_product_acquisition_vendor_idx` (`vendor_id` ASC),
  INDEX `fk_product_acquisition_product_idx` (`product_id` ASC),
  INDEX `date_index` (`date` ASC),
  CONSTRAINT `fk_product_acquisition_vendor`
    FOREIGN KEY (`vendor_id`)
    REFERENCES `symfony`.`vendor` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_product_acquisition_product`
    FOREIGN KEY (`product_id`)
    REFERENCES `symfony`.`product` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB;
````

> Use Integer types for money. You may be tempted to use database's money type or floating point numbers to store money values, don't. At best you are coupling yourself to a specific database and/or a specific platform. At worst you may have precision leacks that may be extremly hard to debug.

- contact table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`contact` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(45) NULL DEFAULT NULL,
  `email` VARCHAR(45) NULL DEFAULT NULL,
  `mobile_phone` VARCHAR(45) NULL DEFAULT NULL,
  `phone` VARCHAR(45) NULL DEFAULT NULL,
  PRIMARY KEY (`id`))
ENGINE = InnoDB;
````

- account table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`account` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `login` VARCHAR(45) NULL DEFAULT NULL,
  `password` VARCHAR(100) NULL DEFAULT NULL,
  `email` VARCHAR(45) NULL DEFAULT NULL,
  `active` TINYINT(4) NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  INDEX `login_index` (`login` ASC),
  INDEX `password_index` (`password` ASC))
ENGINE = InnoDB;
````

- customer table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`customer` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `account_id` INT(10) UNSIGNED NOT NULL,
  `contact_id` INT(10) UNSIGNED NOT NULL,
  PRIMARY KEY (`id`),
  INDEX `fk_customer_account_idx` (`account_id` ASC),
  INDEX `fk_customer_contact_idx` (`contact_id` ASC),
  CONSTRAINT `fk_customer_account`
    FOREIGN KEY (`account_id`)
    REFERENCES `symfony`.`account` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_customer_contact`
    FOREIGN KEY (`contact_id`)
    REFERENCES `symfony`.`contact` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8
COLLATE = utf8_general_ci;
````

- country table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`country` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `code` VARCHAR(5) NULL DEFAULT NULL,
  `name` VARCHAR(45) NULL DEFAULT NULL,
  PRIMARY KEY (`id`))
ENGINE = InnoDB;
````

- address table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`address` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(45) NULL DEFAULT NULL,
  `city` VARCHAR(45) NULL DEFAULT NULL,
  `street_name` VARCHAR(45) NULL DEFAULT NULL,
  `street_number` VARCHAR(45) NULL DEFAULT NULL,
  `building` VARCHAR(45) NULL DEFAULT NULL,
  `entrance` VARCHAR(45) NULL DEFAULT NULL,
  `number` VARCHAR(45) NULL DEFAULT NULL,
  `deleted` TINYINT(4) NULL DEFAULT 0,
  `country_id` INT(10) UNSIGNED NOT NULL,
  PRIMARY KEY (`id`),
  INDEX `fk_address_country_idx` (`country_id` ASC),
  CONSTRAINT `fk_address_country`
    FOREIGN KEY (`country_id`)
    REFERENCES `symfony`.`country` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB;
````

>**deleted** column will be used to achive what is called a *soft delete*. That is to update the deleted column value instead of issuing a DELETE query.

- a customer can have many addresses

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`customer_has_address` (
  `customer_id` INT(10) UNSIGNED NOT NULL,
  `address_id` INT(10) UNSIGNED NOT NULL,
  PRIMARY KEY (`customer_id`, `address_id`),
  INDEX `fk_customer_has_address_address_idx` (`address_id` ASC),
  INDEX `fk_customer_has_address_customer_idx` (`customer_id` ASC),
  CONSTRAINT `fk_customer_has_address_customer`
    FOREIGN KEY (`customer_id`)
    REFERENCES `symfony`.`customer` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_customer_has_address_address`
    FOREIGN KEY (`address_id`)
    REFERENCES `symfony`.`address` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB;

````

- order table

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`order` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `status` SMALLINT(5) UNSIGNED NULL DEFAULT NULL,
  `create_date` DATETIME NULL DEFAULT NULL,
  `customer_id` INT(10) UNSIGNED NOT NULL,
  PRIMARY KEY (`id`),
  INDEX `fk_order_customer_idx` (`customer_id` ASC),
  CONSTRAINT `fk_order_customer`
    FOREIGN KEY (`customer_id`)
    REFERENCES `symfony`.`customer` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8
COLLATE = utf8_general_ci;
````

>Don't add indexes on low cardinality columns. For the moment, we can think of a handful *status* an order can have (new, delivered, cancelled..). We can also have a long sight and think about dozens of possible values. It is very unlinkly however, that an order can have thousands or even hundreds of possible status. Adding an index on this column will increase INSERT and UPDATE queries execution time without noticeably speeding up the SELECT queries.

- an order has many product_sales with different quantities

````sql
CREATE TABLE IF NOT EXISTS `symfony`.`order_product_line` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `order_id` INT(10) UNSIGNED NOT NULL,
  `product_sale_id` INT(10) UNSIGNED NOT NULL,
  `quantity` INT(11) NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  INDEX `fk_order_product_line_order_idx` (`order_id` ASC),
  INDEX `fk_order_product_line_product_sale_idx` (`product_sale_id` ASC),
  CONSTRAINT `fk_order_product_line_order`
    FOREIGN KEY (`order_id`)
    REFERENCES `symfony`.`order` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_order_product_line_product_sale`
    FOREIGN KEY (`product_sale_id`)
    REFERENCES `symfony`.`product_sale` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB;
````
