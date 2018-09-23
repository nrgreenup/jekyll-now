---
layout: post
title: "Examining Employee Data in mySQL (with a dash of Tableau)"
author: "Nolan Greenup"
date: "23/9/2018"
output:
  html_document: default
---

In this post, I discuss a handful of SQL queries on employee data that I've implemented in mySQL Workbench. I wrote these queries from scratch, specifically choosing research questions that would require me to write more complex code that I had relatively less experience working with. 

NOTE: This blog post is a work in progress. I will have more commands (with accompanying discussions) posted on 9/24/2018.

## Contents
_Introductory Information_     
[About the Data](#about-the-data)   

_Queries_   
[Average Salary by Month and Year of Hire](#average-salary)   
[Average Salary of Non-Managers by Department and Position](#non-managerial-salary)   
[Jobs Held by Those in the Top Salary Decile](#high-paying-jobs)   
[Departmental Pay Equality](#pay-equality)   
[Youngest and Oldest Employees Ever Hired](#youngest-and-oldest-employees-hired)   
[Hired or Promoted: How Managers Got the Job](#hired-or-promoted)   
[Number of Subordinates Per Manager](#number-of-subordinates)   
[Number and Percentage of Staffers Promoted](#promoted-staffers)

## About the Data
I use data from a fictional employee database available [here](https://dev.mysql.com/doc/employee/en/). The database includes six tables:
 - employees: employee number, birth date, hire date, first and last name, gender
 - emp_dept: employee number, department number, starting date, ending date
 - departments: department number, department name
 - dept_manager: department number, employee number, start date, end date
 - salaries: employee number, salary, start date, end date
 - titles: employee number, title, start date, end date
 
A visualization of the schema is available [here](https://dev.mysql.com/doc/employee/en/sakila-structure.html).

## Average Salary
```sql
CREATE INDEX idx_hire_date ON employees (hire_date) ; 

 ALTER TABLE employees ADD hire_year INT ;
UPDATE employees SET hire_year = (YEAR(employees.hire_date)) ;

 ALTER TABLE employees ADD hire_month INT ;
UPDATE employees SET hire_month = (MONTH(employees.hire_date)) ;

SELECT m.hire_year as 'Hire Year',
       m.hire_month as 'Hire Month',
       COUNT(m.emp_no) as 'Number of Hires',
       ROUND(AVG(m.start_sal), 2) as 'Average Starting Salary'
  FROM 
     (SELECT e.emp_no, hire_year, 
             hire_month, MIN(salary) AS start_sal
       FROM employees AS e 
            INNER JOIN salaries AS s 
            ON e.emp_no = s.emp_no 
	    GROUP BY e.emp_no) AS m
 GROUP BY hire_year, hire_month 
 ORDER BY hire_year, hire_month ;
 ```

## Non-Managerial Salary
```sql
SELECT dept_name, 
       title, 
       AVG(salary) AS avg_salary
  FROM salaries AS s 
       INNER JOIN dept_emp AS de 
       ON s.emp_no = de.emp_no 
       
       INNER JOIN departments AS d 
       ON de.dept_no = d.dept_no 
       
       INNER JOIN titles AS t 
       ON s.emp_no = t.emp_no     
 WHERE title NOT IN 
                  (SELECT title 
                     FROM titles
                    WHERE title = 'Manager')
 GROUP BY dept_name, title
 ORDER BY avg_salary DESC ;
 ```

## High Paying Jobs
```sql
SELECT title, 
       COUNT(p.salary) as num_emp_90p
  FROM
	   (SELECT title, 
             salary,
             RANK() OVER (ORDER BY salary DESC) as sal_rank
        FROM employees AS e 
             INNER JOIN salaries AS s 
             ON e.emp_no = s.emp_no 
            
             INNER JOIN titles AS t 
             ON e.emp_no = t.emp_no  
	     WHERE s.to_date = '9999-01-01' AND   -- '9999-01-01' in the end date indicates current position    
             t.to_date = '9999-01-01') AS p
 WHERE p.sal_rank <= .1 * 240124   -- 240124 is the total number of current employees
 GROUP BY title 
 ORDER BY num_emp_90p DESC ;
```

## Pay Equality
```sql
SELECT dept_name, 
       gender, 
       AVG(salary) as 'Average Salary'
  FROM employees AS e 
       INNER JOIN dept_emp AS de 
       ON e.emp_no = de.emp_no 
        
       INNER JOIN departments AS d 
       ON de.dept_no = d.dept_no 
        
       INNER JOIN salaries AS s 
       ON e.emp_no = s.emp_no      
 GROUP BY dept_name, gender WITH ROLLUP ;
```

## Youngest and Oldest Employees Hired
```sql
(SELECT 'Youngest Hire' AS Description, 
        first_name, 
        last_name,
        hire_date, 
        birth_date, 
        DATEDIFF(hire_date, birth_date) / 365.25 AS hire_age 
   FROM employees
  ORDER BY hire_age
  LIMIT 1)
UNION 
(SELECT 'Oldest Hire' AS Description, 
        first_name, 
        last_name,
        hire_date, 
        birth_date, 
        DATEDIFF(hire_date, birth_date) / 365.25 AS hire_age 
   FROM employees
  ORDER BY hire_age DESC
  LIMIT 1) ;
```

## Hired or Promoted
```sql
SELECT dept_name, 
       first_name, 
       last_name,
       CASE WHEN dm.from_date = e.hire_date 
            THEN 'Hired'
            ELSE 'Promoted' 
	    END AS how_attained
  FROM dept_manager dm 
       INNER JOIN employees AS e 
       ON dm.emp_no = e.emp_no 
       
       INNER JOIN departments AS d 
       ON dm.dept_no = d.dept_no
 ORDER BY dept_name, last_name, how_attained ;
 ```
 
## Number of Subordinates
```sql
SELECT e.emp_no, 
       e.first_name, 
       e.last_name,
       e.hire_date, 
       ct.num_subordinates
  FROM employees AS e 
       INNER JOIN	
           (SELECT dm.emp_no, 
                   COUNT(de.emp_no) as num_subordinates
              FROM dept_emp AS de, dept_manager AS dm
             WHERE de.dept_no = dm.dept_no AND
                  (dm.from_date <= de.from_date AND dm.to_date >= de.from_date OR
                   dm.from_date BETWEEN de.from_date AND de.to_date)
             GROUP BY dm.emp_no) AS ct 
       ON e.emp_no = ct.emp_no
 ORDER BY num_subordinates DESC ;
 ```
 
 ## Promoted Staffers
 ```sql
SELECT DISTINCT title FROM titles ; 

SELECT 'Promoted Staffers' AS description, 
       COUNT(ps.emp_no) AS num_promoted,
       (SELECT COUNT(ps.emp_no)) / (SELECT COUNT(ts.emp_no) 
                                      FROM
                                          (SELECT t.emp_no,
                                                  COUNT(title) as num_total
                                             FROM titles AS t 
                                                  INNER JOIN employees AS e 
                                                  ON t.emp_no = e.emp_no
                                            WHERE title LIKE '%staff%'
                                            GROUP BY t.emp_no) AS ts) AS pct_promoted
  FROM
     (SELECT t.emp_no, 
             COUNT(title) as num_pos
        FROM titles AS t 
       WHERE title LIKE '%staff%'
       GROUP BY t.emp_no
      HAVING num_pos >= 2) AS ps ;
```
