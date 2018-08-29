---
layout: post
title: "Predicting Customer Churn with Python"
author: "Nolan Greenup"
date: "28/8/2018"
output:
  html_document: default
---

In this post, I examine and discuss the 4 classifiers I fit to predict customer churn: K Nearest Neighbors, Logistic Regression, Random Forest, and Gradient Boosting. I first outline the data cleaning and preprocessing procedures I implemented to prepare the data for modeling. I then proceed to a discusison of each model in turn, highlighting what the model actually does, how I tuned the model's hyperparameters, and the results of each model. The best performing model has an out-of-sample test performance of 80%, and more importantly, an AUC of 0.84. I conclude by comparing the efficacy of the models and discussing avenues by which the models could be improved even further.

NOTE: The following report provides a detailed description of my analyses. To obtain the files used for analysis, see the [repository home page](https://github.com/nrgreenup/sales-forecast).

## Contents
_Introductory Information_     
[About the Data](#about-the-data)   

_Data Cleaning and Analysis_   
[Data Cleaning and Preprocessing](#data-cleaning-and-preprocessing)   
[Model 1: K Nearest Neighbors](#model-1-k-nearest-neighbors)   
[Model 2: Logistic Regression](#model-2-logistic-regression)   

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
Having done so, I can iterate through all of the categorical variables and plot their distributions. Plots of categorical distributions reveal, for instance, that roughly 27% of customers observed in the dataset churned in the past month and that 16% of customers are senior citizens. 
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
![Distribution of Customer Churn]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/customer-churn/plot_dist-Churn.png "Distribution of Customer Churn"){: height="350px" width="350px"}
![Distribution of Senior Citizen]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/customer-churn/plot_dist-SeniorCitizen.png "Distribution of Senior Citizen"){: height="350px" width="350px"}

Next, I perform some analyses exploring the distribution of customer churn broken down by features of interest. For instance, violin plots clearly indicate that the bulk of those who churn are relatively new customers and tend to have higher monthly charges. Looking at categorical predictors, we can also see that, for instance, contract length matters quite a bit; about 43% of month-to-month contracts churn, while only 11% of one-year and 3% of two-year contracts do the same. Visualizations of the distributions of all variables (including those not included here), as well as select breakdowns of churn, can be found [here](https://github.com/nrgreenup/nrgreenup.github.io/tree/master/images/customer-churn).

![Distribution of Customer Churn by Tenure and Monthly Charges]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/customer-churn/plot-churn_by_charges_tenure.png "Distribution of Customer Churn by Tenure and Monthly Charges"){: height="500px" width="700px"}
![Distribution of Customer Churn by Contract Length]({{ https://github.com/nrgreenup/nrgreenup.github.io/blob/master/ }}/images/customer-churn/plot-churn_by_contract.png "Distribution of Customer Churn by Contract Length"){: height="500px" width="700px"}

Now, having explored the data thoroughly, I preprocess the data for analysis. First, as mentioned above, it is necessary to standardize continuous measures so that they are on the same scale. I use `StandardScaler` from `sklearn.preprocessing` to do so, transforming each continuous measure to have zero mean and unit standard deviation (sd = 1).
```python
### Standardize continuous variables for distance based models
scale_vars = ['tenure', 'MonthlyCharges']
scaler = StandardScaler() 
df[scale_vars] = scaler.fit_transform(df[scale_vars])
df[scale_vars].describe()
```
Lastly, I binarize all binary variables using `LabelEncoder` from `sklearn.preprocessing` and one-hot encode multi-class categorical variables using `get_dummies` from `pandas`. One-hot encoding is one method of preparing categorical variables so that they can be properly utilized by machine learning algorithms. For any categorical variable with some number of levels *d*, one-hot encoding returns *d* binary variables, one each for every level of the original categorical variable. For instance, encoding a categorical measure of political partisanship (Democrat, Republican, Independent, Other) would return 4 binary variables, one each for Democrat, Republican, Independent, and other.
```python
## Binarize binary variables
df_enc = df.copy()
binary_vars = ['gender', 'SeniorCitizen', 'Partner', 'Dependents',
               'PhoneService', 'MultipleLines', 'OnlineSecurity', 
               'OnlineBackup','DeviceProtection', 'TechSupport', 
               'StreamingTV', 'StreamingMovies', 'PaperlessBilling', 'Churn']
enc = LabelEncoder()
df_enc[binary_vars] = df_enc[binary_vars].apply(enc.fit_transform)

## One-hot encode multi-category cat. variables
multicat_vars = ['InternetService', 'Contract', 'PaymentMethod']
df_enc = pd.get_dummies(df_enc, columns = multicat_vars)
df_enc.iloc[:,16:26] = df_enc.iloc[:,16:26].astype(int)
print(df_enc.info())
```
Having ensured that the data has been fully pre-processed for analysis, the final step before fitting models is to split the data into training and test sets. I use 70% of the observations for training the models and reserve 30% for testing their out-of-sample performance. Because there is some class imbalance in the target variable -- only 1/4 of customers churned -- I stratify my train-test split on the target variable. This ensures that roughly 1/4 of the observations in both the training and testing data are customers who have churned.
```python
X = df_enc.drop('Churn', axis = 1)
y= df_enc['Churn']
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size = .3,stratify = y, random_state = 30)
```

## Model 1: K Nearest Neighbors
I begin with one of the simplest, but most interpretable, machine learning models: K Nearest Neighbors (KNN). In short, KNN classifies a given observation *O* based on the *K* observations that are most similar to it using a simple majority vote. For instance, in a binary setting where *K*=9, if 5 of the 9 nearest observations to *O* are "1" and 4 of the 9 nearest observations to *O* are "0", then *O* is classified as "1". "Nearest" is determined using one of a several distance metrics; the `scikit.learn` KNN classifer defaults to [Euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance).

I implement the KNN using the following code. In addition, I implement grid searching to tune two hyperparameters of the model, *K* and weights. This allows us to find the most optimal value of for both hyperparameters in order to maximize performance. Grid searching is often computationally taxing, so while I use it here for the KNN, computational performance could be sped up using randomized searching (more on that later)! I also use 5-fold [cross-validation](https://en.wikipedia.org/wiki/Cross-validation_(statistics)) to help reduce overfitting the model as much as possible.
```
knn = KNeighborsClassifier()

## Set up hyperparameter grid for tuning
knn_param_grid = {'n_neighbors' : np.arange(5,26),
                  'weights' : ['uniform', 'distance']}

## Tune hyperparameters
knn_cv = GridSearchCV(knn, param_grid = knn_param_grid, cv = 5)

## Fit knn to training data
knn_cv.fit(X_train, y_train)

## Get info about best hyperparameters
print("Tuned KNN Parameters: {}".format(knn_cv.best_params_))
print("Best KNN Training Score:{}".format(knn_cv.best_score_)) 

## Predict knn on test data
print("KNN Test Performance: {}".format(knn_cv.score(X_test, y_test)))

## Obtain model performance metrics
knn_pred_prob = knn_cv.predict_proba(X_test)[:,1]
knn_auroc = roc_auc_score(y_test, knn_pred_prob)
print("KNN AUROC: {}".format(knn_auroc))
knn_y_pred = knn_cv.predict(X_test)
print(classification_report(y_test, knn_y_pred))

## OUTPUT 
Tuned KNN Parameters: {'n_neighbors': 23, 'weights': 'uniform'}
Best KNN Training Score: 0.7949290060851927
KNN Test Performance: 0.7818267865593942
KNN AUROC: 0.8227581684032563
```
The grid search selects *K* = 23 and uniform weights; thus, each observation is classified according to the 23 observations nearest to it, with each of those 23 observations having equal weight in the final classification. Because the best training score during cross-validation (0.795) is close to our out-of-sample test performance (0.782), we can be reasonably confident we did not overfit our model. The area under the receiver operating characteristics curve (AUROC) for the KNN is 0.823. An AUC of 1 indicates a perfect model; an AUC of 0.5 indicates a model that performs no better than a random guess. Thus, we want a model with an AUC as close to 1 as possible. The obtained AUC of 0.823 is strong for a first model! 

## Model 2: Logistic Regression
I next implement a logistic regression classifier.
