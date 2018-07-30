---
layout: post
title: "Fun with MySQL"
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
_INSERT IMAGE HERE (GROSS SALES)_

We see that Margaret, Janet, and Nancy have made the most sales, while Steven rounds out the table as the lowest performing salesperson. Of course, we may be interested in net sales that take into account, for instance, discounts customers may have received on their purchase. Because this information is also available in the `orderDET` relation, a minor change to the `SELECT` statement takes care of this for us. In the output screenshotted below, there is little change in the rank ordering of employees, with only Laura and Robert swapping ranks, but the value of the sales expectedly decreases a bit.

```SQL
SELECT   LastName, FirstName, COUNT(O.OrderID) AS NumOrders, 
         SUM(UnitPrice*(1-Discount)*Quantity) AS SalesNet
FROM     employees E INNER JOIN orders O ON (E.EmployeeID = O.EmployeeID) 
         INNER JOIN orderDET D ON (O.OrderID = D.OrderID)
GROUP BY E.EmployeeID
ORDER BY SalesNet DESC;
```
_INSERT IMAGE HERE (NET SALES)_

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
_insert image of not null results here_

I next turn to information in the customers table that could potentially decrease operational costs significantly for Northwind. Seeing that Northwind is a global operation, it would be desirable to reduce shipping costs as much as possible. For instance, we may want to ensure that all orders shipped to the same area are shipped together on the same vessel. Thus, knowing which customers are located near one another could help significantly reduce shipping costs for Northwind. Using a series of self joins, we find all pairs of 5 companies that are located in the same city. Results show that there are 7 combinations of 5-customer pairs that are located within the same city.

```SQL
SELECT C1.City, C1.CompanyName, C2.CompanyName, C3.CompanyName, C4.CompanyName, C5.CompanyName
FROM   customers C1, customers C2, customers C3, customers C4, customers C5
WHERE  C1.City = C2.City AND C2.City = C3.City AND C3.City = C4.City AND C4.City = C5.City 
       AND C1.CompanyName < C2.CompanyName AND C2.CompanyName < C3.CompanyName
       AND C3.CompanyName < C4.CompanyName AND C4.CompanyName < C5.CompanyName;
```

_insert image of 5 city customers here_

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
 
 _10% savings screenshot_
