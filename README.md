# pyopls - Orthogonal Projection to Latent Structures in Python. 
[![Build Status](https://travis-ci.org/BiRG/pyopls.svg?branch=master)](https://travis-ci.org/BiRG/pyopls)

This package provides a scikit-learn-style transformer to perform OPLS.
OPLS is a pre-processing method to remove variation from the descriptor 
variables that are orthogonal to the target variable (1).

This package also provides a class to validate OPLS models using a 
1-component PLS regression with cross-validation and permutation tests (2)
for both regression and classification metrics (from permutations of the
target) and feature PLS loadings (from permutations of the features).

## Installation
pyopls is available via [pypi](https://pypi.org/project/pyopls/):
```shell
pip install pyopls
```

## Notes:
* The implementation provided here is equivalent to that of the 
  [libPLS MATLAB library](http://libpls.net/), which is a faithful
  recreation of Trygg and Wold's algorithm.
  *   This package uses a different definition for R<sup>2</sup>X, however (see
      below)
* `OPLS` inherits `sklearn.base.TransformerMixin` (like
  `sklearn.decomposition.PCA`) but does not inherit 
  `sklearn.base.RegressorMixin` because it is not a regressor like
  `sklearn.cross_decomposition.PLSRegression`. You can use the output of
  `OPLS.transform()` as an input to another regressor or classifier.
* Like `sklearn.cross_decomposition.PLSRegression`, `OPLS` will center
  both X and Y before performing the algorithm. This makes centering by
  class in PLS-DA models unnecessary.
* The `score()` function of `OPLS` performs the R<sup>2</sup>X score, the
  ratio of the variance in the transformed X to the variance in the
  original X. A lower score indicates more orthogonal variance removed.
* `OPLS` only supports 1-column targets.

## Examples
### Perform OPLS and PLS-DA on wine dataset
OPLS-processed data require only 1 PLS component. Performing a
4-component OPLS improves accuracy from 95% to 100% and DQ<sup>2</sup> (3) from 0.76
to 0.84.
```python
import pandas as pd
import numpy as np
from pyopls import OPLS
from sklearn.datasets import load_wine
from sklearn.cross_decomposition import PLSRegression
from sklearn.model_selection import cross_val_predict, LeaveOneOut
from sklearn.metrics import r2_score, accuracy_score

wine_data = load_wine()
df = pd.DataFrame(wine_data['data'], columns=wine_data['feature_names'])
df['classification'] = wine_data['target']
df = df[df.classification.isin((0, 1))]
target = df.classification.apply(lambda x: 1 if x else -1)  # discriminant for class 1 vs class 0
X = df[[c for c in df.columns if c != 'classification']]
opls = OPLS(4)
Z = opls.fit_transform(X, target)

pls = PLSRegression(1)
y_pred = cross_val_predict(pls, X, target, cv=LeaveOneOut())
q_squared = r2_score(target, y_pred)  # 0.733
dq_squared = r2_score(target, np.clip(y_pred, -1, 1))  # 0.759
accuracy = accuracy_score(target, np.sign(y_pred))  # 0.954

processed_y_pred = cross_val_predict(pls, Z, target, cv=LeaveOneOut())
processed_q_squared = r2_score(target, processed_y_pred)  # 0.836
processed_dq_squared = r2_score(target, np.clip(processed_y_pred, -1, 1))  # 0.838
processed_accuracy = accuracy_score(target, np.sign(processed_y_pred))  # 1.0

r2_X = opls.score(X)  # 0.0000320 (most variance is removed)
``` 

### Validation
The `fit()` method of `OPLSValidator` will find the optimum number of
components to remove, then evaluate the results on a 1-component
`sklearn.cross_decomposition.PLSRegression` model. A permutation test is
performed for each metric by permuting the target and for the PLS
loadings by permuting the features.
 
This snippet will determine the best number of components to remove then
plot the ROC curves of the classifiers on processed and unprocessed data
and plot the PLS and OPLS scores for the samples.

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from pyopls import OPLSValidator
from sklearn.datasets import load_wine
from sklearn.cross_decomposition import PLSRegression
from sklearn.model_selection import cross_val_predict, LeaveOneOut
from sklearn.metrics import roc_curve, roc_auc_score

wine_data = load_wine()
df = pd.DataFrame(wine_data['data'], columns=wine_data['feature_names'])
df['classification'] = wine_data['target']
df = df[df.classification.isin((0, 1))]
target = df.classification.apply(lambda x: 1 if x else -1)  # discriminant for class 1 vs class 0
X = df[[c for c in df.columns if c!='classification']]

validator = OPLSValidator(k=-1).fit(X, target)

Z = validator.opls_.transform(X)
pls = PLSRegression(1)
y_pred = cross_val_predict(pls, X, target, cv=LeaveOneOut())
processed_y_pred = cross_val_predict(pls, Z, target, cv=LeaveOneOut())

fpr, tpr, thresholds = roc_curve(target, y_pred)
roc_auc = roc_auc_score(target, y_pred)
proc_fpr, proc_tpr, proc_thresholds = roc_curve(target, processed_y_pred)
proc_roc_auc = roc_auc_score(target, processed_y_pred)

plt.figure(0)
plt.plot(fpr, tpr, lw=2, color='blue', label=f'Unprocessed (AUC={roc_auc:.4f})')
plt.plot(proc_fpr, proc_tpr, lw=2, color='red',
         label=f'{validator.n_components_}-component OPLS (AUC={proc_roc_auc:.4f})')
plt.plot([0, 1], [0, 1], color='gray', lw=2, linestyle='--')
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate')
plt.title('ROC Curve')
plt.legend(loc='lower right')
plt.show()

plt.figure(1)
df['t'] = validator.pls_.x_scores_
df['t_ortho'] = validator.opls_.T_ortho_[:, 0]
pos_df = df[target==1]
neg_df = df[target==-1]
plt.scatter(neg_df['t'], neg_df['t_ortho'], c='blue', label='Class 0')
plt.scatter(pos_df['t'], pos_df['t_ortho'], c='red', label='Class 1')
plt.title('PLS Scores')
plt.xlabel('t_ortho')
plt.ylabel('t')
plt.legend(loc='lower right')
plt.show()
```
#### ROC Curve
![roc curve](roc_curve.png) 
#### Scores Plot
![scores plot](scores.png)
## References
1. Johan Trygg and Svante Wold. Orthogonal projections to latent structures (O-PLS).
   *J. Chemometrics* 2002; 16: 119-128. DOI: [10.1002/cem.695](https://dx.doi.org/10.1002/cem.695)
2. Eugene Edington and Patrick Onghena. "Calculating P-Values" in *Randomization tests*, 4th edition.
   New York: Chapman & Hall/CRC, 2007, pp. 33-53. DOI: [10.1201/9781420011814](https://doi.org/10.1201/9781420011814).
3. Johan A. Westerhuis, Ewoud J. J. van Velzen, Huub C. J. Hoefsloot, Age K. Smilde. Discriminant Q-squared for 
   improved discrimination in PLSDA models. *Metabolomics* 2008; 4: 293-296. 
   DOI: [10.1007/s11306-008-0126-2](https://doi.org/10.1007/s11306-008-0126-2)
