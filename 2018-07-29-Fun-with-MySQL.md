---
layout: post
title: "Fun with MySQL"
author: "Nolan Greenup"
date: "29/7/2018"
output:
  html_document: default
---

I've been focusing recently on trying to take my SQL skills to the next level -- defining more complicated subqueries, working with triggers and views, Online Analytical Processing tools, and the like. After taking six (exceptionally taught) [online courses](https://lagunita.stanford.edu/courses/DB/2014/SelfPaced/about) and working through numerous excercises, I wanted to get my hands dirty with some data in a more applied way. In this post, I walk through some queries and analyses I performed on the oft-used [Northwind](https://www.alphasoftware.com/documentation/pages/GettingStarted/GettingStartedTutorials/Basic%20Tutorials/Northwind/northwindMySQL.xml)
data. All of the analyses herein were done in MySQL using the GUI MySQL Workbench.

## Contents
_Introductory Information_     
[Data Description](#data-description)   

_Analyses & Queries_   
[Preliminary Analyses of Relations](#preliminary-analyses-of-relations)    
[Employee Performance](#employee-performance)   
[Customer Analytics](#customer-analytics)   
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
SELECT LastName, FirstName, COUNT(O.OrderID) AS NumOrders, 
       SUM(UnitPrice*(1-Discount)*Quantity) AS SalesNet
FROM employees E INNER JOIN orders O ON (E.EmployeeID = O.EmployeeID) 
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
Next, a query using aggregation seen in prior queries gives us what we want to see... TO BE CONTINUED!!!
