[![PyPI](https://img.shields.io/pypi/v/edaspy)](https://pypi.python.org/pypi/EDAspy/)
[![PyPI license](https://img.shields.io/pypi/l/EDAspy.svg)](https://pypi.python.org/pypi/EDAspy/)
[![Downloads](https://pepy.tech/badge/edaspy)](https://pepy.tech/project/edaspy)
[![Documentation Status](https://readthedocs.org/projects/edaspy/badge/?version=latest)](https://edaspy.readthedocs.io/en/latest/?badge=latest)

# EDAspy

## Description

In this package some Estimation of Distribution Algorithms (EDAs) are implemented. EDAs are a type of evolutionary algorithms. Depending on the type of EDA, different dependencies among the variables can be considered.

Three EDAs have been implemented:
* Binary univariate EDA. It can be used as a simple example of EDA, or to use it for feature selection.
* Continuous univariate EDA. It can be used for hyper-parameter optimization.
* Continuous multivariate EDA. Using Gaussian Bayesian Networks to model an abstract representation of the search space.
* Continuous multivariate EDA. Another approach not using GBNs that can be used for hyper-parameter optimization, or other problems.
* Binary multivariate EDA. This approach selects the best time series transformation to improve the model forecasting performance.

The following codes are some easy examples of how the functions should be used. In order to see some real examples, I recommend to see the Notebooks examples where some Jupyter notebooks are presented.

## Examples

#### Binary univariate EDA
It can be used as a simple example of EDA, or to use it for feature selection. The cost function to optimize is the metric of the model. An example is shown.

In the Notebooks directory there is a example.

```python
from EDAspy.optimization.univariate import EDA_discrete as EDAd
import pandas as pd

def check_solution_in_model(dictionary):
    MAE = prediction_model(dictionary)
    return MAE

vector = pd.DataFrame(columns=['param1', 'param2', 'param3'])
vector.loc[0] = 0.5

EDA = EDAd(MAX_IT=200, DEAD_ITER=20, SIZE_GEN=30, ALPHA=0.7, vector=vector, 
            cost_function=check_solution_in_model, aim='minimize')

bestcost, solution, history = EDA.run(output=True)
print(bestcost)
print(solution)
print(history)
```

The example is an implementation for feature selection (FS) for a prediction model (prediction_model). prediction_model depends on some variables. The prediction model, receives a dictionary with keys 'param_N' (N from 1 to number of parameters) and values 1 or 0 depending if that variables should be included or not. The model returns a MAE which we intend to minimize.

The EDA receives as input the maximum number of iterations, the number of iterations with no best global cost improvement, the generations size, the percentage of generations to select as best individuals, the initial vector of probabilities for each variable to be used, the cost functions to optimize, and the aim ('minimize' or 'maximize') of the optimizer.

Vector probabilities are usually initialized to 0.5 to start from an equilibrium situation. EDA returns the best cost found, the best combination of variables, and the history os costs found to be plotted.

#### Continuous univariate EDA

This EDA is used when some continuous parameters must be optimized. 

In the Notebooks directory there is a example.

```python
from EDAspy.optimization.univariate import EDA_continuous as EDAc
import pandas as pd
import numpy as np

wheights = [20,10,-4]

def cost_function(dictionary):
    function = wheights[0]*dictionary['param1']**2 + wheights[1]*(np.pi/dictionary['param2']) - 2 - wheights[2]*dictionary['param3']
    if function < 0:
        return 9999999
    return function

vector = pd.DataFrame(columns=['param1', 'param2', 'param3'])
vector['data'] = ['mu', 'std', 'min', 'max']
vector = vector.set_index('data')
vector.loc['mu'] = [5, 8, 1]
vector.loc['std'] = 20
vector.loc['min'] = 0
vector.loc['max'] = 100

EDA = EDAc(SIZE_GEN=40, MAX_ITER=200, DEAD_ITER=20, ALPHA=0.7, vector=vector, 
            aim='minimize', cost_function=cost_function)
bestcost, params, history = EDA.run()
print(bestcost)
print(params)
print(history)
```

In this case, the aim is to optimize a cost function which we want to minimize. The three parameters to optimize are continuous. This EDA must be initialized with some initial values (mu), and an initial range to search (std). Optionally, a minimum and a maximum can be specified.

As in the binary EDA, the best cost found, the solution and the cost evolution is returned.

#### Continuous multivariate EDA

In this case, dependencies among the variables are considered and managed with a Gaussian Bayesian Network. This EDA must be initialized with historical records in order to try to find the optimum result. A parameter (beta) is included to control the influence of the historical records in the final solution. Some of the variables can be evidences (fixed values for which we want to find the optimum of the other variables). 
The optimizer will find the optimum values of the non-evidence-variables based on the value of the evidences. This is widely used in problems where dependencies among variables must be considered.

In the Notebooks directory there is a example.

```python
from EDAspy.optimization.multivariate import EDA_multivariate as EDAm
import pandas as pd

blacklist = pd.DataFrame(columns=['from', 'to'])
aux = {'from': 'param1', 'to': 'param2'}
blacklist = blacklist.append(aux, ignore_index=True)

data = pd.read_csv(path_CSV)  # columns param1 ... param5
evidences = [['param1', 2.0],['param5', 6.9]]

def cost_function(dictionary):
    return dictionary['param1'] + dictionary['param2'] + dictionary['param3'] + dictionary['param4'] + dictionary['param5']

EDAm = EDAm(MAX_ITER=200, DEAD_ITER=20, data=data, ALPHA=0.7, BETA=0.4, cost_function=cost_function,
                 evidences=evidences, black_list=blacklist, n_clusters=6, cluster_vars=['param1', 'param5'])
output = EDAm.run(output=True)

print('BEST', output.best_cost_global)
```
This is the most complex EDA implemented. Bayesian networks are used to represent an abstraction of the search space of each iteration, where new individuals are sampled. As a graph is a representation with nodes and arcs, some arcs can be forbidden by the black list (pandas dataframe with the forbidden arcs). 

In this case the cost function is a simple sum of the parameters. The evidences are variables that have fixed values and are not optimized. In this problem, the output would be the optimum value of the parameters which are not in the evidences list.
Due to the evidences, to help the structure learning algorithm to find the arcs, a clustering by the similar values is implemented. Thus, the number of clusters is an input, as well as the variables that are considered in the clustering.

In this case, the output is the self class that can be saved as a pickle in order to explore the attributes. One of the attributes is the optimum structure of the optimum generation, from which the structure can be plotted and observe the dependencies among the variables. The function to plot the structure is the following:
```python
from EDAspy.optimization.multivariate import print_structure
print_structure(structure=structure, var2optimize=['param2', 'param3', 'param4'], evidences=['param1', 'param5'])
```

![Structure praph plot](/structure.PNG "Structure of the optimum generation found by the EDA")

#### Another Continuous multivariate EDA approach

In this EDA approach, new individuals are sampled from a multivariate normal distribution. Evidences are not allowed in the optimizer. If desired, the previous approach should be used.
The EDA is initialized, as in the univariate continuous EDA, with univariate mus and sigma for the variables. In the execution, a multivariate gaussian is built to sample from it. As it is multivariate, correlation among variables is considered.

In the Notebooks directory there is a example.

```python
import pandas as pd
from EDAspy.optimization.multivariate import EDA_multivariate_gaussian as EDAmg


def cost_function(dictionary):
    suma = dictionary['param1'] + dictionary['param2']
    if suma < 0:
        return 999999999
    return suma

mus = pd.DataFrame(columns=['param1', 'param2'])
mus.loc[0] = [10, 8]

sigma = pd.DataFrame(columns=['param1', 'param2'])
sigma.loc[0] = 5

EDAmulti = EDAmg(SIZE_GEN=40, MAX_ITER=1000, DEAD_ITER=50, ALPHA=0.6, aim='minimize',
                                     cost_function=cost_function, mus=mus, sigma=sigma)

bestcost, params, history = EDAmulti.run(output=True)
print(bestcost)
print(params)
print(history)
```

The cost function to optimize is the minimization of two parameter sum. Both parameters are continuous, and to be initialized two pandas dataframes are needed: one with mus and another with sigmas (diagonal of the covariance matrix)

The EDA returns the best cost, the combination and the history of costs if wanted to be plotted.

#### Time series transformations selection

When working with large datasets in time series Machine Learning projects, it is common to use different time series transformations in order to improve the forecasting model. In this approach an EDA is used to find the optimum transformations of the time series.

In the Notebooks directory there is a example.

## Getting started

#### Prerequisites
R must be installed to use the "optimization.multivariate" subpackage, with the following installed libraries: c("bnlearn", "dbnR", "data.table")
To manage R from python, rpy2 package must also be installed.

#### Installing
```
pip install EDAspy
```
