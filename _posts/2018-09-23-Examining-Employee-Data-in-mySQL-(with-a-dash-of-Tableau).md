---
layout: post
title: "Examining Employee Data in mySQL (with a dash of Tableau)"
author: "Nolan Greenup"
date: "23/9/2018"
output:
  html_document: default
---

In this post, I discuss a handful of SQL queries on employee data that I've implemented in mySQL Workbench. I wrote these queries from scratch, specifically choosing research questions that would require me to write more complex code that I had relatively less experience working with. 


## Contents
_Introductory Information_     
[About the Data](#about-the-data)   

_Data Cleaning and Analysis_   
[Data Cleaning and Preprocessing](#data-cleaning-and-preprocessing)   
[Model 1: K Nearest Neighbors](#model-1-k-nearest-neighbors)   
[Model 2: Logistic Regression](#model-2-logistic-regression)      
[Model 3: Random Forest](#model-3-random-forest)   
[Model 4: Stochastic Gradient Boosting](#model-4-stochastic-gradient-boosting)   

## About the Data
I use data from a fictional employee database available [here](https://dev.mysql.com/doc/employee/en/).

