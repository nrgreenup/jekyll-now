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
### Import required packages for preprocessing and analysis
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
### Import data                  
df = pd.read_csv('churn.csv')
```
I then dive into some exploratory data analysis. I start with as broad a scope as possible, looking at generic summary information. The following commands indicate that there are 7043 observations over 21 features,most of which are categorical. The scales of our two continuous features are quite different from each other; customer tenure has a mean of 32 and standard devation of 25, while monthly charges has a mean of 65 and standard deviation of 30. Becuase some of our classifiers are distance-based (e.g. K Nearest Neighbors), we will need to standardize these features to be on the same scale. I do so later.
```python
### Get preliminary info about dataframe
print(df.info()) 
print(df.isnull().sum()) 
print(df.describe()) 
```
I next collapse a handful of the categorical variables related to specific internet services from multiclass to binary. All customers that don't have a specialized internet service because they don't have internet service at all are recoded. The same is done for the feature measuring whether the customer has multiple phone lines; if they do not have any phone service at all, their value for multiple phone lines is changed from 'no phone service' to 'no'.
```python
# 6 features, convert 'no internet service' to 'no'
no_int_service_vars = ['OnlineSecurity', 'OnlineBackup', 
                       'DeviceProtection','TechSupport', 
                       'StreamingTV', 'StreamingMovies']
                       
for var in no_int_service_vars:
    df[var] = df[var].map({'No internet service': 'No',
                           'Yes': 'Yes',
                           'No': 'No'}).astype('category')
    
for var in no_int_service_vars:
    print(df[var].value_counts())
```
Having done so, I can iterate through all of the categorical variables and plot their distributions.
```python
## Define list of categorical variables
cat_vars = ['gender', 'SeniorCitizen', 'Partner', 'Dependents',
            'PhoneService', 'MultipleLines', 'InternetService', 
            'OnlineSecurity', 'OnlineBackup','DeviceProtection', 'TechSupport', 
            'StreamingTV', 'StreamingMovies', 'Contract', 'PaperlessBilling',
            'PaymentMethod', 'Churn']
            
## Plot distributions of categorical variables
for var in cat_vars:
    ax = sns.countplot(x = df[var], data = df, palette = 'colorblind')
    total = float(len(df[var])) 
    for p in ax.patches:
        height = p.get_height()
        ax.text(p.get_x()+p.get_width()/2.,
                height + 10,
                '{:1.2f}'.format(height/total),
                ha="center")
    plt.title('Distribution of ' + str(var))
    plt.ylabel('Number of Customers')
    plt.figtext(0.55, -0.01 , 
                'Decimal above bar is proportion of total in that class',
                horizontalalignment = 'center', fontsize = 8,
                style = 'italic')
    plt.xticks(rotation = 60)
    plt.tight_layout()
    plt.savefig('plot_dist-' + str(var) + '.png', dpi = 200)
    plt.show()
```
