---
layout:     post
title:      "Stacking with Bayesian Optimization"
subtitle:   ""
date:       2017-11-08 07:00:00
author:     "Siqi"
header-img: "img/default.jpg"
mathjax: true
tags:
    - python
    - machine learning
---

## Introduction

Stacking has become a common practice in machine learning competitions such as Kaggle. Here is a simple stacking template based on the model I used in Alibab Tianchi competition. This template incorporates bayesian optimization, which will save you the time spent on grid searching hyper-parameters. I have uploaded my template notebook [here](https://github.com/Shiki-H/stacking-template/blob/master/Stacking%20Template.ipynb).   

As a disclaimer, since I am using random data generated by ```sklearn```, the performance of the stacking model may vary. Again, since this is just for illustration, I copied the parameter range from my original model, which may not be optimal. In addition, to obtain decent results from bayesian optimization, you should increase the number of iterations and combine with your experience when setting the range for hyper-parameters.  

```python
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
import warnings
warnings.filterwarnings('ignore')
%matplotlib inline
pd.set_option('display.max_columns', None)
```

## Load Data


```python
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.cross_validation import KFold
```

```python
# generate random samples for inllustration
# we will work with an imbalanced data set 
X, y = make_classification(n_samples=5000, 
                           n_classes=2, 
                           n_features=10, 
                           n_informative=5, 
                           weights=[0.95, 0.05],
                           random_state=42)
```


```python
x_train, x_test, y_train, y_test = \
                train_test_split(X, y, random_state=42, test_size=0.2)
```

## Stacking Preparations


```python
# Set up parameters
ntrain = x_train.shape[0]
ntest = x_test.shape[0]
SEED = 42 
NFOLDS = 5 
kf = KFold(ntrain, n_folds= NFOLDS, random_state=SEED)
```

There are quite a few ways to train meta-features for stacking model. I have tried a few and eventually decided to stay with the one mentioned by [Anisotropdc](https://www.kaggle.com/arthurtok/introduction-to-ensembling-stacking-in-python/), which is to compute predictions on test set using base model trained on each fold and then take average. Based on my experience, this method gives the best performance comparing to other simple stacking methods. If you are interested in fancier methods, you may want to take a look [here](https://link.springer.com/content/pdf/10.1023%2FB%3AMACH.0000015881.36452.6e.pdf). 


```python
# Function to obtain out-of-fold prediction
def get_oof(clf, x_train, y_train, x_test):
    oof_train = np.zeros((ntrain,))
    oof_test = np.zeros((ntest,))
    oof_test_skf = np.empty((NFOLDS, ntest))

    for i, (train_index, test_index) in enumerate(kf):
        x_tr = x_train[train_index]
        y_tr = y_train[train_index]
        x_te = x_train[test_index]

        clf.fit(x_tr, y_tr)

        oof_train[test_index] = clf.predict(x_te)
        oof_test_skf[i, :] = clf.predict(x_test)

    oof_test[:] = oof_test_skf.mean(axis=0)
    return oof_train.reshape(-1, 1), oof_test.reshape(-1, 1)
```

## Tuning Base Models


```python
from sklearn.model_selection import cross_val_score
from sklearn.metrics import f1_score

# bayesian optimization
from bayes_opt import BayesianOptimization

# base models
from sklearn.ensemble import RandomForestClassifier
from lightgbm import LGBMClassifier
from sklearn.svm import LinearSVC
from sklearn.neural_network import MLPClassifier

# second layer model
from xgboost import XGBClassifier
```

For large data sets, tuning each base model can take a very long time. To prevent any accidental failure, I recommend saving the models (or to be more specific, the parameters of the models) immediately when the tuning is done. 

To save and load your model, you can use sklearn's ```joblib```.  
```python
# save model
joblib.dump(model, filename)

# load model
joblib.load(filename)
```


```python
from sklearn.externals import joblib
from time import gmtime, strftime
import json
```


```python
# parameters for bayesian optimization
gp_params = {"alpha": 1e-5}
n_iter=5
```

```python
# Function to generate file name based completion time
def generate_filename(model):
    return str(model)+strftime("%Y-%m-%d_%H:%M", gmtime())+'.pkl'

# Function to save the log of parameter tuning
def BO_log(BO, name):
    fname = name + strftime("%Y-%m-%d_%H:%M", gmtime()) + '.json'
    with open(fname, 'w') as fp:
        json.dump(BO.res, fp)
    print(fname + ' saved')
```

#### Random Forest

Since we are dealing with imbalanced data, we need to be careful in model training. In the past, my colleagues and I have tried every popular sampling methods (over-sampling, under-sampling, and SMOTE) but in the end we decided to adjust the class weight directly. There are a few advantages of this method: 1) it is easy to do; 2) it does not make any assumptions on the distribution of the features; 3) when it is not clear what weight you should assign to each class, you can treat it as a hyper-parameter and let machine do the work for you.  

Regarding the metrics used for optimization, I am using f1 score since this was the judging criteria of the competition I participated in. Of course you can use any metric you wish, but keep in mind that when you are dealing imbalanced data, you have to be careful with your choice. 


```python
def rfc(n_estimators, min_samples_split, max_depth, class_weight):
    return RandomForestClassifier(n_estimators=int(n_estimators),
            min_samples_split=int(min_samples_split),
            max_depth=int(max_depth),
            random_state=42,
            n_jobs=-1,
            bootstrap=False,
            class_weight={0:1, 1:class_weight}
        )
def rfccv(**params):
    val = cross_val_score(
        rfc(**params),
        x_train, y_train, scoring='f1', cv=5
    ).mean()
    return val
```


```python
rfcBO = BayesianOptimization(
        rfccv,
        {'n_estimators': (50, 500),
        'min_samples_split': (2, 25),
        'max_depth': (3, 12),
        'class_weight': (4, 10)
        }
    )
```

```python
rfcBO.maximize(n_iter=n_iter, **gp_params)
```

```python
BO_log(rfcBO, 'rfc')
```

```python
rf_clf = rfc(**rfcBO.res['max']['max_params'])
```


```python
joblib.dump(rf_clf, generate_filename('rf_clf'))
```

```python
rf_clf.fit(x_train, y_train)
rf_pred = rf_clf.predict(x_test)
print(f1_score(y_test, rf_pred))
```

```python
rf_oof_train, rf_oof_test = get_oof(rf_clf, x_train, y_train, x_test)
```

#### LightGBM 


```python
def lgb(n_estimators, learning_rate, scale_pos_weight):
    return LGBMClassifier(n_estimators=int(n_estimators),
            learning_rate=learning_rate,
            scale_pos_weight=scale_pos_weight,
            seed=42
        )
def lgbcv(**params):
    val = cross_val_score(
        lgb(**params),
        x_train, y_train, scoring='f1', cv=5
    ).mean()
    return val
```


```python
lgbcBO = BayesianOptimization(
        lgbcv,
        {'n_estimators': (50, 500),
         'learning_rate': (0.01, 0.1),
         'scale_pos_weight': (4, 10)
        }
    )
```


```python
lgbcBO.maximize(n_iter=n_iter, **gp_params)
```

```python
BO_log(lgbcBO, 'lgb')
```

```python
lgb_clf = lgb(**lgbcBO.res['max']['max_params'])
```


```python
joblib.dump(lgb_clf, generate_filename('lgb_clf'))
```

```python
lgb_clf.fit(x_train, y_train)
lgb_pred = lgb_clf.predict(x_test)
print(f1_score(y_test, lgb_pred))
```

```python
lgb_oof_train, lgb_oof_test = get_oof(lgb_clf, x_train, y_train, x_test) 
```

#### SVM

We need to normalize data first before training SVM and MLP. 


```python
from sklearn.preprocessing import StandardScaler
```


```python
x_train_scaled = StandardScaler().fit_transform(x_train)
x_test_scaled = StandardScaler().fit_transform(x_test)
```


```python
def linsvc(C, class_weight):
    return LinearSVC(C=C,
            random_state=42,
            class_weight={0:1, 1:class_weight}
        )
def linsvccv(**params):
    val = cross_val_score(
        linsvc(**params),
        x_train_scaled, y_train, scoring='f1', cv=5, n_jobs=-1
    ).mean()
    return val
```


```python
linsvcBO = BayesianOptimization(
        linsvccv,
        {'C': (1, 10000),
         'class_weight': (4, 10)}
    )
```


```python
linsvcBO.maximize(n_iter=n_iter, **gp_params)
```

```python
BO_log(linsvcBO, 'linsvc')
```

```python
linsvc_clf = linsvc(**linsvcBO.res['max']['max_params'])
```


```python
joblib.dump(linsvc_clf, generate_filename('linsvc_clf'))
```

```python
linsvc_clf.fit(x_train_scaled, y_train)
linsvc_pred = linsvc_clf.predict(x_test_scaled)
print(f1_score(y_test, linsvc_pred))
```

```python
linsvc_oof_train, linsvc_oof_test = get_oof(linsvc_clf,x_train_scaled, y_train, x_test_scaled)
```

#### MLP


```python
def mlp(first_layer, second_layer, alpha):
    return MLPClassifier(
            hidden_layer_sizes=(int(first_layer), int(second_layer)),
            alpha=10**alpha
        )
def mlpccv(**params):
    val = cross_val_score(
        mlp(**params),
        x_train_scaled, y_train, scoring='f1', cv=5, n_jobs=-1
    ).mean()
    return val
```


```python
mlpcBO = BayesianOptimization(
        mlpccv,
        {'first_layer': (4, 8),
         'second_layer': (2, 4),
         'alpha': (-5, -2)}
    )
```


```python
mlpcBO.maximize(n_iter=n_iter, **gp_params)
```

```python
BO_log(mlpcBO, 'mlp')
```

```python
mlp_clf = mlp(**mlpcBO.res['max']['max_params'])
```


```python
joblib.dump(mlp_clf, generate_filename('mlp_clf'))
```

```python
mlp_clf.fit(x_train_scaled, y_train)
mlp_pred = mlp_clf.predict(x_test_scaled)
print(f1_score(y_test, mlp_pred))
```

```python
mlp_oof_train, mlp_oof_test = get_oof(mlp_clf, x_train_scaled, y_train, x_test_scaled) 
```

## Stacking Base Models


```python
base_predictions_train = pd.DataFrame(
    {'RandomForest': rf_oof_train.ravel(),
     'LightGBM': lgb_oof_train.ravel(),
     'LinSVC': linsvc_oof_train.ravel(),
     'MLP': mlp_oof_train.ravel(),
    })
base_predictions_train.head()
```

<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>LightGBM</th>
      <th>LinSVC</th>
      <th>MLP</th>
      <th>RandomForest</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
</div>




```python
sns.heatmap(base_predictions_train.astype(float).corr().values,
            xticklabels=base_predictions_train.columns,
            yticklabels=base_predictions_train.columns,
            annot=True)
```

![png](/img/in-post/stacking-template/output_58_1.png)


This heat map was taken from one of the randomly generated data I have run. In general, as mentioned by [Džeroski and Ženko](https://link.springer.com/content/pdf/10.1023%2FB%3AMACH.0000015881.36452.6e.pdf), it is a good idea to include heterogenous classifiers as base models for stacking, i.e. classifiers that use different model representations. Since both random forest and lightgbm are tree-based, their high correlation is expected. 


```python
xg_train = np.concatenate((rf_oof_train, 
                           lgb_oof_train, 
                           linsvc_oof_train, 
                           mlp_oof_train, 
                          ), axis=1)
xg_test = np.concatenate((rf_oof_test, 
                          lgb_oof_test, 
                          linsvc_oof_test, 
                          mlp_oof_test, 
                         ), axis=1)
```


```python
def xgb(max_depth, 
          learning_rate, 
          n_estimators, 
          gamma, 
          min_child_weight, 
          subsample,
          scale_pos_weight):
    return XGBClassifier(
            max_depth=int(max_depth),
            learning_rate=learning_rate,
            n_estimators=int(n_estimators),
            min_child_weight=int(min_child_weight),
            gamma=max(gamma, 0),
            subsample=max(min(subsample, 1), 0),
            scale_pos_weight=scale_pos_weight,
            random_state=42,
            n_jobs=-1
        )
def xgbcv(**params):
    val = cross_val_score(
        xgb(**params),
        xg_train, y_train, scoring='f1', cv=5
    ).mean()
    return val
```


```python
xgbBO = BayesianOptimization(
        xgbcv,
        {'max_depth': (3, 10),
         'learning_rate': (0.05, 0.3),
         'n_estimators':(50, 500),
         'min_child_weight':(1, 20),
         'gamma':(0, 10),
         'subsample':(0.5, 1),
         'scale_pos_weight':(2, 10)
        }
    )
```


```python
xgbBO.maximize(n_iter=n_iter, **gp_params)
```

```python
BO_log(xgbBO, 'xgb')
```

```python
xgb_clf = xgb(**xgbBO.res['max']['max_params'])
```


```python
joblib.dump(xgb_clf, generate_filename('xgb_clf'))
```

```python
xgb_clf.fit(xg_train, y_train)
xgb_pred = xgb_clf.predict(xg_test)
print(f1_score(y_test, xgb_pred))
```

## Conclusion

The aim of this template is to provide a quick set up of a stacking ensemble with bayesian optimization. Based on my work experience in fraud detection, finding the right features is much more important than building a fancy model. I believe that for most machine learning applications, the key to create the best model is thorough understanding of the problem so that one could figure out features that best describes the situation. I hope this template can save you some time in getting the model to work so that you can stay focused on the problem per se. 