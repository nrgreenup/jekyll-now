---
layout: post
title: "An Exploration of Northwind Using MySQL"
author: "Nolan Greenup"
date: "29/7/2018"
output:
  html_document: default
---

I've been focusing recently on trying to take my SQL skills to the next level -- defining more complicated subqueries, working with triggers and views, Online Analytical Processing tools, and the like. After taking six (exceptionally taught) [online courses](https://lagunita.stanford.edu/courses/DB/2014/SelfPaced/about) and working through numerous excercises, I wanted to get my hands dirty with some data in a more applied way. In this post, I walk through some queries and analyses I performed on the oft-used [Northwind](https://www.alphasoftware.com/documentation/pages/GettingStarted/GettingStartedTutorials/Basic%20Tutorials/Northwind/northwindMySQL.xml)
data. All of the analyses herein were done in MySQL using the MySQL Workbench GUI.

## Contents
_Introductory Information_     
[Data Description](#data-description)   

_Analyses & Queries_   
[Preliminary Analyses of Relations](#preliminary-analyses-of-relations)    
[Employee Performance](#employee-performance)   
[Sales Analytics](#sales-analytics)     
[A Few Useful Triggers](#a-few-useful-triggers)

## Data Description
The Northwind database contains sales data about a fictitious company. The database contains 13 tables (relations), six of which I use for the analyses discussed later: 
   + categories (types of products sold)
   + customers (information about Northwind's customers)
   + employees (information about Northwind's employees)
   + orders (contains order information)
   + order details (additional details about orders)
   + products (details about specific products)
   
## Preliminary Analyses of Relations
Before jumping into any more complicated queries, it useful to get an initial picture of what each relation contains. After changing the name of the "order details" relation to "orderDET" for easier referencing, I get to work.

The preliminary analyses give a good overview of the data. There are 77 distinct products and 9 product categories. There are 91 customers who have collectively placed 830 complete orders. In some of those orders, customers purchased multiple products; there are 2155 product orders. Lastly, there are 9 employees, 5 of whom operate in the United States and 4 in the United Kingdom. With this basic information in hand, I dive into some analyses.
```SQL 
ALTER TABLE `northwind`.`order details` 
RENAME TO  `northwind`.`orderDET`;

SELECT * FROM categories; 
SELECT * FROM customers;
SELECT * FROM employees;
SELECT * FROM orderDET limit 3000;
SELECT * FROM orders limit 3000;
SELECT * FROM products;
```

## Employee Performance
With regard to analysis, I'll briefly look at 3 sectors of Northwind's operation: employees, customers, and sales. I start with the former. I decide to examine two aspects of employee performance that would be of use to organizational management: individual performance and aggregate performance by age. Considering the former, human resources decisions (e.g. raises, promotions, termination) are made easier when performance metrics are available. We may, then, be interested in identifying the best and worst performing salespeople at Northwind. 

I first create an index on the `EmployeeID` attribute in the `employees` relation. Indexes on an attribute are particularly helpful for increasing efficiency when that attribute is either used as a criterion in a selection condition (`WHERE` clauses) or is part of a joining condition. Here, because the `employees` relation is joined to the `orders` relation using the `EmployeeID` attribute, an index is helpful. Of course, the increased efficiency is minimal on a Northwind-like database of relatively small size; the true benefit of indexes is seen in much larger databases.

A brief note: The `NumOrders` attribute displays the number of product orders, which we found to be 2155 earlier. So if one customer bought 4 different products, that would count as 4 in the `NumOrders` attribute, even if the products were all bought during one transaction.
```SQL
CREATE UNIQUE INDEX EmployeeID on employees(EmployeeID);

SELECT LastName, FirstName, COUNT(O.OrderID) AS NumOrders, 
       SUM(UnitPrice*Quantity) AS SalesGross
FROM   employees E INNER JOIN orders O ON (E.EmployeeID = O.EmployeeID) 
       INNER JOIN orderDET D ON (O.OrderID = D.OrderID)
GROUP BY E.EmployeeID
ORDER BY SalesGross DESC;
```

![Employee Gross Sales]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/northwind-mySQL/employee-gross_sales.png "Employee Gross Sales"){: height="400px" width="400px"}

We see that Margaret, Janet, and Nancy have made the most sales, while Steven rounds out the table as the lowest performing salesperson. Of course, we may be interested in net sales that take into account, for instance, discounts customers may have received on their purchase. Because this information is also available in the `orderDET` relation, a minor change to the `SELECT` statement takes care of this for us. In the output screenshotted below, there is little change in the rank ordering of employees, with only Laura and Robert swapping ranks, but the value of the sales expectedly decreases a bit.
```SQL
SELECT   LastName, FirstName, COUNT(O.OrderID) AS NumOrders, 
         SUM(UnitPrice*(1-Discount)*Quantity) AS SalesNet
FROM     employees E INNER JOIN orders O ON (E.EmployeeID = O.EmployeeID) 
         INNER JOIN orderDET D ON (O.OrderID = D.OrderID)
GROUP BY E.EmployeeID
ORDER BY SalesNet DESC;
```

![Employee Net Sales]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/northwind-mySQL/employee-net_sales.png "Employee Net Sales"){: height="400px" width="400px"}

Lastly, we may be interested in some aggregate measure of employee performance. Though sociodemographic data is sparse in the `employees` relation, there is information about employees' date of birth. Given this, I decided to investigate whether there is a difference in sales performance between "younger" and "older" employees. To do so, I decided to compare the sales of all employees born before 1960 with those born during or after 1960. First, I created an indicator variable based on the recorded date of birth:
```SQL
ALTER TABLE employees ADD birth1960 TEXT;
UPDATE employees SET birth1960 = 'pre' WHERE EmployeeID <> 0 AND BirthDate < '1960-01-01 00:00:00';
UPDATE employees SET birth1960 = 'post' WHERE EmployeeID <> 0 AND BirthDate >= '1960-01-01 00:00:00';
```

Next, a query using aggregation seen in prior queries gives us what we want. Approximately 62% of net sales are made by salespeople born before 1960. Of course, there are 5 salespeople in the the pre-1960 group, but only 4 in the post-1960 group. Nevertheless, the pre-1960 workers still outperform their younger counterparts; the average sales for those born before 1960 is about $157.5K, while average sales for those born during or after 1960 is roughly $119.5K.
```SQL
SELECT birth1960, COUNT(O.OrderID) AS NumOrders, 
       SUM(UnitPrice*(1-Discount)*Quantity) AS SalesNet
FROM employees E INNER JOIN orders O ON (E.EmployeeID = O.EmployeeID) 
     INNER JOIN orderDET D ON (O.OrderID = D.OrderID)
GROUP BY E.birth1960
ORDER BY SalesNet DESC;
```

## Sales Analytics 
Next, I run a few queries that provide some insight into Northwind's customers and sales. I begin by finding all customers and all of the orders those customers, returning: Customer name and ID, order ID and date, the country the order is shipped to, and the employee ID of the salesperson who completed the transaction. 
```SQL 
SELECT C.CustomerID, C.CompanyName, O.OrderID, O.OrderDate, O.ShipCountry, O.EmployeeID
FROM   customers C LEFT JOIN orders O ON (C.CustomerID = O.CustomerID);
```

Of course, this can easily be rewritten as a `RIGHT JOIN` by flipping the `orders` relation to the left side of the `JOIN` and the `customers` relation to the right side, per the preference of the end user. The results are equivalent.
```SQL
SELECT C.CustomerID, C.CompanyName, O.OrderID, O.OrderDate, O.ShipCountry, O.EmployeeID
FROM   orders O RIGHT JOIN customers C ON (C.CustomerID = O.CustomerID);
```

The earlier `SELECT * FROM orders limit 3000;` command found that there were 830 separate orders placed, though again some of those orders contained multiple products. But here, left (or right) joining the `customers` and `orders` relations provides 832 results. This indicates that there are 2 customers that are in the database that have not yet placed an order. Perhaps Northwind is invested in fostering an ongoing relationship with these customers. Luckily, they are easily identifiable. By appending an `IS NOT NULL` selection condition, the below code finds that the customers with IDs FISSA and PARIS have not placed any orders.
```SQL
SELECT C.CustomerID, C.CompanyName, O.OrderID, O.OrderDate
FROM   customers C LEFT JOIN orders O ON (C.CustomerID = O.CustomerID)
WHERE  OrderID IS NULL;
```

![Customers with No Orders]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/northwind-mySQL/customers-no_orders.png "Customers with No Orders"){: height="400px" width="400px"}

I next turn to information in the customers table that could potentially decrease operational costs significantly for Northwind. Seeing that Northwind is a global operation, it would be desirable to reduce shipping costs as much as possible. For instance, we may want to ensure that all orders shipped to the same area are shipped together on the same vessel. Thus, knowing which customers are located near one another could help significantly reduce shipping costs for Northwind. Using a series of self joins, we find all pairs of 5 companies that are located in the same city. Results show that there are 7 combinations of 5-customer pairs that are located within the same city.
```SQL
SELECT C1.City, C1.CompanyName, C2.CompanyName, C3.CompanyName, C4.CompanyName, C5.CompanyName
FROM   customers C1, customers C2, customers C3, customers C4, customers C5
WHERE  C1.City = C2.City AND C2.City = C3.City AND C3.City = C4.City AND C4.City = C5.City 
       AND C1.CompanyName < C2.CompanyName AND C2.CompanyName < C3.CompanyName
       AND C3.CompanyName < C4.CompanyName AND C4.CompanyName < C5.CompanyName;
```

![Pairs of 5 Customers from Same City]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/northwind-mySQL/customers-five_in_city.png "Pairs of 5 Customers from Same City"){: height="400px" width="400px"}

In my final set of queries, I want to discover what companies are the "big savers" when it comes to discounts. I first begin by finding all of those companies that have, on average, saved more than 10% on their orders. The following code does just that. Of course, it should be noted there are a few different ways this query could be written, both with and without subqueries in the `FROM` clause. Both queries below produce the same results. There are 8 companies that have saved more than 10% on average on their orders.
```SQL
SELECT * 
FROM (SELECT C.CompanyName,
             SUM(UnitPrice*Quantity) AS SalesGross , 
             SUM(UnitPrice*(1-Discount)*Quantity) AS SalesNet,
             1 - SUM(UnitPrice*(1-Discount)*Quantity) / SUM(UnitPrice*Quantity) AS Discount
      FROM orders O INNER JOIN orderDET D ON (O.OrderID = D.OrderID) 
           INNER JOIN customers C ON (O.CustomerID = C.CustomerID)
      GROUP BY O.CustomerID
      HAVING Discount > 0.1
      ORDER BY Discount DESC ) S ;
 
SELECT C.CompanyName,
       SUM(UnitPrice*Quantity) AS SalesGross , 
       SUM(UnitPrice*(1-Discount)*Quantity) AS SalesNet,
       1 - SUM(UnitPrice*(1-Discount)*Quantity) / SUM(UnitPrice*Quantity) AS Discount
FROM   orders O INNER JOIN orderDET D ON (O.OrderID = D.OrderID) 
       INNER JOIN customers C ON (O.CustomerID = C.CustomerID)
GROUP BY O.CustomerID
HAVING Discount > 0.1
ORDER BY Discount DESC;
 ```
 
 ![Customers with 10% Savings]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/northwind-mySQL/customers-10_percent_disc.png "Customers with 10% Savings"){: height="400px" width="400px"}

Of course, we could complicate the matter further by adding additional conditions. For instance, let's say we wanted to know all of the customers who averaged more than 5% savings on their orders that are located in a city which houses another Northwind customer. To do so, we add a subquery into the `WHERE` clause, which finds only those companies that share a city in common with another company. We find that there are 6 companies that save 5% on average that are in a city with another Northwind customer.
```SQL
SELECT * 
FROM(
	SELECT C.CompanyName,
               C.City,
               SUM(UnitPrice*Quantity) AS SalesGross , 
               SUM(UnitPrice*(1-Discount)*Quantity) AS SalesNet,
               1 - sum(UnitPrice*(1-Discount)*Quantity) / SUM(UnitPrice*Quantity) AS Discount
	FROM orders O INNER JOIN orderDET D ON (O.OrderID = D.OrderID) 
             INNER JOIN customers C ON (O.CustomerID = C.CustomerID)
        GROUP BY O.CustomerID
        HAVING Discount > 0.05
        ORDER BY Discount DESC ) S 
WHERE CompanyName IN 
		(SELECT DISTINCT C1.CompanyName
		FROM  customers C1, customers C2, customers C3
		WHERE C1.City = C2.City AND C2.City = C3.City AND
		      C1.CompanyName <> C2.CompanyName AND 
                      C2.CompanyName <> C3.CompanyName AND 
                      C1.CompanyName <> C3.CompanyName) ;  
```

![Customers with 5% Savings in City with Another Customer]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/northwind-mySQL/customers-5_percent_disc_shared_city.png "Customers with 5% Savings in City with Another Customer"){: height="400px" width="400px"}

Though less efficient, we could also implement the same query by changing the `WHERE` subquery to the following, which uses the `UNION` operator:
```SQL
WHERE CompanyName IN 
   (SELECT C1.CompanyName
   FROM customers C1, customers C2, customers C3
   WHERE C1.City = C2.City AND C2.City = C3.City 
         AND C1.CompanyName < C2.CompanyName AND C2.CompanyName < C3.CompanyName 
UNION
   SELECT C2.CompanyName
   FROM customers C1, customers C2, customers C3
   WHERE C1.City = C2.City AND C2.City = C3.City 
         AND C1.CompanyName < C2.CompanyName AND C2.CompanyName < C3.CompanyName 
UNION
   SELECT C3.CompanyName
   FROM customers C1, customers C2, customers C3
   WHERE C1.City = C2.City AND C2.City = C3.City 
         AND C1.CompanyName < C2.CompanyName AND C2.CompanyName < C3.CompanyName ) ;
```

Of course, there are so many other interesting queries we could run on the Northwind data, a few of which I've included below. I won't go into any of them in great detail, but they each provide other interesting insights.
``` SQL
/* IN WHAT COUNTRY ARE SALES HIGHER? */
SELECT Country, COUNT(O.OrderID) AS NumOrders, 
       SUM(UnitPrice*(1-Discount)*Quantity) AS SalesNet
FROM employees E INNER JOIN orders O ON (E.EmployeeID = O.EmployeeID) 
     INNER JOIN orderDET D ON (O.OrderID = D.OrderID)
GROUP BY E.Country
ORDER BY SalesNet DESC;

/* BREAKDOWN BY CATEGORIES */
SELECT C.CategoryID, C.CategoryName, SUM(D.UnitPrice*(1-D.Discount)*D.Quantity) AS SalesNet
FROM orderDET D INNER JOIN products P ON (D.ProductID = P.ProductID) 
     INNER JOIN categories C ON(C.CategoryID = P.CategoryID)
GROUP BY C.CategoryID   
ORDER BY SalesNet DESC;

/* FIND ALL PRODUCTS THAT HAVE GENERATED GREATER THAN 25K IN SALES */
SELECT P.ProductID, ProductName, SUM(D.UnitPrice*(1-D.Discount)*D.Quantity) AS SalesNet
FROM orderDET D INNER JOIN products P ON (D.ProductID = P.ProductID)
GROUP BY P.ProductID    
HAVING SalesNet > 25000     
ORDER BY SalesNet DESC;
```

## A Few Useful Triggers
Lastly, when examining the schema of the Northwind data, it is readily apparent that a few triggers would be useful in maintaining efficiency and database integrity. First, I consider creating a trigger that prevents insertion of negative values that are required to be non-negative. For instance, it would never make sense for the unit price of an order to be negative. Yet, inserting a negative unit price is allowed by the database: `INSERT INTO orderDet VALUES (99999, 99999, -5, 99999 , 0)` is executed without error. Looking at the `orderDET` attribute, we see the tuple has been inserted.

![Negative Value for Unit Price in Order Details Table]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/northwind-mySQL/trigger-orderDETneg.png "Negative Value for Unit Price in Order Details Table"){: height="400px" width="400px"}
 
 This is allowed because the only constraint placed on the unit price attribute is that it cannot be null, as seen in the code that created the `orderDET` relation:
 ```SQL
 DROP TABLE IF EXISTS `order details`;
CREATE TABLE `order details` (
  `OrderID` int(11) NOT NULL,
  `ProductID` int(11) NOT NULL,
  `UnitPrice` decimal(19,4) NOT NULL DEFAULT '0.0000',
  `Quantity` int(11) NOT NULL DEFAULT '1',
  `Discount` float NOT NULL DEFAULT '0',
  PRIMARY KEY (`OrderID`,`ProductID`),
  KEY `OrderID` (`OrderID`),
  KEY `ProductID` (`ProductID`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
```

What we can do, then, is implement a trigger that raises an error if a user tries to insert a negative unit price:
```SQL
DELIMITER $$
CREATE TRIGGER NegPrice
BEFORE INSERT ON orderDET 
FOR EACH ROW 
BEGIN
	IF New.UnitPrice < 0 THEN
    SIGNAL SQLSTATE '22003' SET MESSAGE_TEXT = 'Unit Price Must Be Positive';
    END IF;
END$$
DELIMITER ;
```
If we try `INSERT INTO orderDet VALUES (99999, 99999, 99999, -5 , 0);` once more, we get the expected error message, and the improper tuple is not inserted.

![Trigger returns error]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/northwind-mySQL/trigger-orderDETerror.png "Trigger Returns Error"){: height="400px" width="400px"}

Of course, we could do the same with any other value we want to restrict to be positive. For instance, product quantity:
```SQL
DELIMITER $$
CREATE TRIGGER NegQuantity
BEFORE INSERT ON orderDET 
FOR EACH ROW 
BEGIN
	IF New.Quantity < 0 THEN
    SIGNAL SQLSTATE '22003' SET MESSAGE_TEXT = 'Quantity Must Be Positive';
    END IF;
END$$
DELIMITER ;
```

Lastly, I implement a trigger that fires when an order is deleted from the `orders` relation. A tuple in the `orderDET` relation only makes sense if it corresponds to a matching tuple in the `orders` relation. Thus, if we delete an order from the database, we should also delete the details of that order. The following trigger does this:
```SQL
DELIMITER $$
CREATE TRIGGER OrderDel
AFTER DELETE ON orders 
FOR EACH ROW 
BEGIN
	DELETE FROM orderDET
    WHERE Old.OrderID = orderDET.OrderId ;
END$$
DELIMITER ;
```

To test that the trigger works properly, I implement the following. Sure enough, the values I inserted into the `orderDET` relation are deleted automatically when the matching tuple in the `orders` relation is deleted. 
```SQL
INSERT INTO orders VALUES(99999, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL);
INSERT INTO orderDET VALUES(99999, 99999, 99999, 99999, 0);
DELETE FROM orders WHERE OrderID = 99999;
```
