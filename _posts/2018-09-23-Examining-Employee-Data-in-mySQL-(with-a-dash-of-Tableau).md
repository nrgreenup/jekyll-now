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

_Data Cleaning and Analysis_   


## About the Data
I use data from a fictional employee database available [here](https://dev.mysql.com/doc/employee/en/). The database includes six tables:
 - employees: employee number, birth date, hire date, first and last name, gender
 - emp_dept: employee number, department number, starting date, ending date
 - departments: department number, department name
 - dept_manager: department number, employee number, start date, end date
 - salaries: employee number, salary, start date, end date
 - titles: employee number, title, start date, end date
A visualization of the schema is available [here](https://dev.mysql.com/doc/employee/en/sakila-structure.html).

