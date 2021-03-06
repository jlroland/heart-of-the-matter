# The Heart of the Matter

## Introduction

<p>Cardiovascular disease continues to be the leading cause of death in the U.S.  Nearly half of all heart attacks have symptoms so mild that individuals don't know they are having a heart attack--a so-called silent heart attack.  Health professionals estimate that 8 to 11 million people suffer from silent strokes <img align="right" width="240" height="130" src="img/marcelo-leal-k7ll1hpdhFA-unsplash.jpg"> each year in which individuals are asymptomatic but would have evidence of a stroke on an MRI.  These risks, combined with the ever-increasing cost of healthcare in the U.S., indicate a need for increased diagnostic efficiency.  How can we identify the individuals who are most at risk?  What preventative measures could be implemented to decrease risk?</p>

## Data

NHANES (National Health and Nutrition Examination Survey) is a national survey conducted by the CDC every couple of years.  The survey contains over 1,000 variables spread across dozens of files, asking questions about lifestyle and medical history, as well as conducting brief medical examinations and running blood tests.
The data used in building this model was taken from the 2015-2016 survey (the most recent available at the time) and can be found at:  
https://wwwn.cdc.gov/nchs/nhanes/continuousnhanes/default.aspx?BeginYear=2015.

There were 16 files used in compiling the data; these were stored in an AWS S3 bucket. The graphic below represents the amount of missing data (shown as white space) encountered in the original compilation (9971 observations, 521 features).
![Missing data before cleaning](img/missing_before.png)

The data was limited to the adult population who participated in both the questionnaire and examination portions of the survey, resulting in 5,735 individuals.  Of the many variables available for analysis, the list was narrowed to produce 63 features for use in evaluating different models.  The elimination process was based first on suspected ranking of relevance to heart disease and second on quality of data (e.g. features with 90% of values missing).  The elimination of features is due mainly to current time constraints; additional work in the future would include attempts to reincorporate eliminated features.  After initial data cleaning based on survey response coding and skip patterns, the amount of missing data decreased significantly (see below).

![Missing data after cleaning](img/missing_after.png)

The remaining missing values were replaced using the KNNImputer from scikit-learn.

Individuals in the dataset were labeled as high-risk for cardiovascular disease based on either:

1. A combination of answers to the Cardiovascular Health questionnaire which indicate symptoms of angina, or
2. Answering the Medical History questionnaire in the affirmative for history of coronary heart disease, angina, heart attack or stroke 
Note: The Cardiovascular Health questionnaire was only administerd to adults age 40+.  The Medical History questionnaire was only administerd to adults age 20+.

## EDA & Getting Started

Initial EDA showed that age and gender would be a good starting point for a baseline model upon which to build.

<p align="center">
  <img src="img/age_gender_dist.png" alt="risk distribution by age & gender">
</p>

The baseline model (using age and gender) was a logistic regression without normalization and the AUC score was 0.85.  About 10% of individuals in the dataset were labeled high-risk; due to class imbalance, only soft classification was used.  Given the unusually high AUC score based on two features, it was worth consulting a confusion matrix (shown below at a probability threshold of 0.5).

<p align="center">
  <img src="img/cf_base_model.png" alt="confusion matrix for baseline model">
</p>

The confusion matrix shows the inclination of the model to predict the negative class, highlighting the class imbalance.

Once a baseline was set, the following models were explored using normalized data for all 63 features:

1. Logistic Regression with L2 regularization
2. Random Forest Classifier with n_estimators=1000 and min_samples_split=10
3. Gradient Boosting Classifier with n_estimators=1000 and max_depth=2

All models performed similarly based on typcial metrics like AUC score.  Logistic regression appeared to have the best predictive ability by a narrow margin.

![ROC curves for applied models](img/roc_comparison.png)

As previously observed, class imbalance had an effect on AUC score.  Therefore, confusion matrices for each model at a threshold of 0.5 are shown below.

<p align="center">
  <img src="img/cf_log.png" alt="logistic regression confusion matrix" width="250" height="250"><img src="img/cf_rf.png" alt="random forest confusion matrix"          width="250" height="250"><img src="img/cf_gbc.png" alt="gradient boosting confusion matrix" width="250" height="250">
</p>

The false negative rate indicated that implementing this model would be impractical due to the high cost associated with false negatives.  The model needed greater ability to predict the high-risk label.

## Mitigating Class Imbalance

Based on the discovery that the false negative rate needed to be decreased, it became apparent that increasing recall would be more valuable than increasing AUC score.  To mitigate the effect of class imbalance, the training data was subjected to oversampling of the minority class before fitting models again. The following models produced improvement in recall:

1. Logistic Regression with L1 regularization using normalized data
2. Random Forest Classifier with n_estimators=1000 and min_samples_split=10
3. Gradient Boosting Classifier with n_estimators=1000 and max_depth=2

Each model showed an increase in log loss, and the AUC scores showed little change from the first round of models.  Each of the models showed improvement in reducing false negatives (see confusion matrices below), but logistic regression still performed best.

<p align="center">
  <img src="img/cf_log_upsample.png" alt="upsampled logistic regression confusion matrix" width="250" height="250"><img src="img/cf_rf_upsample.png" alt="upsampled    random forest confusion matrix" width="250" height="250"><img src="img/cf_gbc_upsample.png" alt="upsampled gradient boosting confusion matrix" width="250"          height="250">
</p>

The plot below shows the beta coefficients for the top features resulting from the upsampled logistic regression.

<p align="center">
  <img src="img/feature_importance_reduced.png" alt="top features by beta coefficient">
</p>

From the list of top features, several were chosen to create a model with a reduced number of features.  The idea behind limiting the number of features was to create a predictive model that could be developed into a user-friendly application.  Incorporating all 63 features would require dozens of inputs from each user before rendering a prediction.  Reducing the number of user inputs by focusing on the top features would reduce the burden on the user with little change to the metrics of the model.  The confusion matrix below represents a logistic regression model trained on a limited number of features.

<p align="center">
  <img src="img/cf_log_upsample_limited.png" alt="confusion matrix for upsampled limited logistic regression model">
</p>

Oversampling the minority class and training the model on the top features increased recall from 0.19 to 0.81.  The corresponding trade-off was a decrease in precision from 0.54 to 0.28.  These results were based on a probability threshold of 0.5; lowering the threshold would produce a greater increase in recall and a greater decrease in precision.

## What I Learned

<img align="left" width="200" height="120" src="img/element5-digital-OyCl7Y4y0Bk-unsplash.jpg">Data quality is the first underlying concern.  Even when surveys are highly-structured and planned carefully, there is plenty of room for error and missing values.  Survey data was used in building this model because it is publicly available, but I suspect the model could be improved by using data from targeted medical research (which is usually restricted from public use).  In addition, data from a longitudinal study would be more beneficial in capturing the long-term effects of individuals' habits.

Class imbalance is the enemy and must be destroyed!  The first models I trained were inclined to predict that no individuals were high-risk.  Implementing a resampling scheme when imbalance exists, whether it's undersampling the majority, oversampling the minority or SMOTE, will greatly benefit the training of the model.

Be mindful of the practical application of the model.  This helps you judge which metrics to use in evaluating potential models--in this case, recall was more important than log loss or AUC because false negatives are especially costly.  It will also guide you in deciding what trade-offs are acceptable.  In this case, health professionals would need to be involved in deciding the acceptable threshold at which the false negatives are low enough to justify the corresponding increase in false positives; the costs associated with these types of error are not equivalent.

### Check out the website for your own prediction

Click [here](http://44.232.41.241) to visit the website.
