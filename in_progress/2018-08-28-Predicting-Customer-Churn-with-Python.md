---
layout: post
title: "Predicting Customer Churn with Python"
author: "Nolan Greenup"
date: "28/8/2018"
output:
  html_document: default
---

In this post, I examine and discuss the 5 classifiers I fit to predict customer churn: K Nearest Neighbors, Logistic Regression, Support Vector Machine, Random Forest, and Gradient Boosting. I first outline the data cleaning and preprocessing procedures I implemented to prepare the data for modeling. I then proceed to a discusison of each model in turn, highlighting what the model actually does, how I tuned the model's hyperparameters, and the results of each model. The best performing model has an accuracy over 80%, coupled with an AUC of roughly 0.85. I conclude by comparing the efficacy of the models and discussing avenues by which the models could be improved even further.

NOTE: The following report provides a detailed description of my analyses. To obtain the files used for analysis, see the [repository home page](https://github.com/nrgreenup/sales-forecast).

## Contents
_Introductory Information_     
[About the Data](#about-the-data)   

_Data Cleaning and Analysis_   
[Data Cleaning and Preprocessing](#data-cleaning-and-preprocessing) 

_Concluding Remarks and Information_   
[Summary of Findings](#summary-of-findings)   

## About the Data
Data for these analyses come from [IBM](https://community.watsonanalytics.com/wp-content/uploads/2015/03/WA_Fn-UseC_-Telco-Customer-Churn.csv). The target feature is customer churn, which is a binary feature "1" if the customer left the company within the last month and "0" if they did not leave in the last month. We have 20 features at the outset that can be used to predict churn, including: whether the customer was a senior citizen, whether they have a partner or dependents, how long they had been customers of the company, monthly charges and other payment information, and whether or not they had several different products through the company (e.g. phone, internet).

## Data Cleaning and Preprocessing
I begin by importing all of the packages necessary for data cleaning and analysis and then import the data and store it to the object `df`.
```python
import pandas as pd
import numpy as np
import seaborn as sns
import matplotlib.pyplot as plt
from sklearn.preprocessing import StandardScaler, LabelEncoder
from sklearn.neighbors import KNeighborsClassifier
from sklearn.model_selection import (train_test_split, GridSearchCV, 
                                     RandomizedSearchCV)
from sklearn.metrics import (roc_auc_score, roc_curve, classification_report)
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.ensemble import (GradientBoostingClassifier,
                              RandomForestClassifier)
                              
df = pd.read_csv('churn.csv')
```
