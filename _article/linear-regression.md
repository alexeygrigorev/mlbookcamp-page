---

title: Linear Regression
layout: article
math: true
draft: true
---

## Normal Equation

$$w = X^T X$$

Where
* $w$ is the weights vector
* $X$ is the feature matrix

code

```python
def linear_regression(X, y):
    # adding the dummy column
    ones = np.ones(X.shape[0]) # A
    X = np.column_stack([ones, X]) # B

    # normal equation formula
    XTX = X.T.dot(X) # C
    XTX_inv = np.linalg.inv(XTX) # D
    w = XTX_inv.dot(X.T).dot(y) # E

    return w[0], w[1:] # F
```

## Regularization

$$w = X^T X$$
