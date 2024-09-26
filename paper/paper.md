---
title: 'TorchSurv: A Lightweight Package for Deep Survival Analysis'
tags:
  - Python
  - Deep Learning
  - Survival Analysis
  - PyTorch
authors:
  - name: Mélodie Monod
    orcid: 0000-0001-6448-2051
    affiliation: 1
  - name: Peter Krusche
    affiliation: 1
  - name: Qian Cao
    affiliation: 2
  - name: Berkman Sahiner
    affiliation: 2
  - name: Nicholas Petrick
    affiliation: 2
  - name: David Ohlssen
    affiliation: 3
  - name: Thibaud Coroller
    orcid: 0000-0001-7662-8724
    corresponding: true
    affiliation: 3
affiliations:
  - name: Novartis Pharma AG, Switzerland
    index: 1
  - name: Center for Devices and Radiological Health, Food and Drug Administration, MD, USA
    index: 2
  - name: Novartis Pharmaceuticals Corporation, NJ, USA
    index: 3
date: 29 July 2024
bibliography: paper.bib
---

# Summary

`TorchSurv` is a Python [package](https://pypi.org/project/torchsurv/) that serves as a companion tool to perform deep survival modeling within the `PyTorch` environment [@paszke2019pytorch]. With its lightweight design, minimal input requirements, full `PyTorch` backend, and freedom from restrictive parameterizations, `TorchSurv` facilitates efficient deep survival model implementation and is particularly beneficial for high-dimensional and complex data scenarios. At its core, `TorchSurv` features calculations of log-likelihoods for prominent survival models (Cox proportional hazards model [@Cox1972], Weibull Accelerated Time Failure (AFT) model [@Carroll2003]) and offers evaluation metrics, including the time-dependent Area Under under the Receiver operating characteristic (ROC) curve (AUC), the Concordance index (C-index) and the Brier Score.
`TorchSurv` has been rigorously [tested](https://opensource.nibr.com/torchsurv/benchmarks.html) using both open-source and synthetically generated survival data, against R and python packages. The package is thoroughly documented and includes illustrative examples. The latest documentation for TorchSurv can be found on our [website](https://opensource.nibr.com/torchsurv/).

# Statement of need

Survival analysis plays a crucial role in various domains, such as medicine, economics or engineering. Thus, sophisticated survival models sugin deep learning opens new opportunities to leverage complex dataset and relationships. However, no existing library provides the flexibility to define the survival model's parameters using a custom `PyTorch`-based neural network.

\autoref{tab:bibliography} compares the functionalities of `TorchSurv` with those of
`auton-survival` [@nagpal2022auton],
`pycox` [@Kvamme2019pycox],
`torchlife` [@torchlifeAbeywardana],
`scikit-survival` [@polsterl2020scikit],
`lifelines` [@davidson2019lifelines], and
`deepsurv` [@katzman2018deepsurv].
Existing libraries constrain users to predefined forms for defining the parameters (e.g., linear function of covariates). While there exist log-likelihood functions in the libraries, they cannot be leveraged.
The limitations on the log-likelihood functions include protected functions (locked model architecture), specialized input requirements (format or class type), and reliance on external libraries like `NumPy` or `Pandas`. Dependence on external libraries hinders automatic gradient calculation within `PyTorch`. Additionally, the implementation of likelihood functions instead of log-likelihood functions, as done by some packages, introduces numerical instability.

![**Survival analysis libraries in Python.** $^1$[@nagpal2022auton], $^{2}$[@Kvamme2019pycox], $^{3}$[@torchlifeAbeywardana], $^{4}$[@polsterl2020scikit], $^{5}$[@davidson2019lifelines], $^{6}$[@katzman2018deepsurv]. A green tick indicates a fully supported feature, a red cross indicates an unsupported feature, a blue crossed tick indicates a partially supported feature. For computing the concordance index, `pycox` requires the use of the estimated survival function as the risk score and does not support other types of time-dependent risk scores. `scikit-survival` does not support time-dependent risk scores in both the concordance index and AUC computation. Additionally, both `pycox` and `scikit-survival` impose the use of inverse probability of censoring weighting (IPCW) for subject-specific weights. `scikit-survival` only offers the Breslow approximation of the Cox partial log-likelihood in case of ties in the event time.\label{tab:bibliography}](table_1.png)

![**Survival analysis libraries in R.** $^1$[@survivalpackage], $^{2}$[@survAUCpackage], $^{3}$[@timeROCpackage], $^{4}$[@risksetROCpackage], $^{5}$[@survcomppackage], $^{6}$[@survivalROCpackage], $^{7}$[@riskRegressionpackage], $^{8}$[@SurvMetricspackage], $^{9}$[@pecpackage]. A green tick indicates a fully supported feature, a red cross indicates an unsupported feature, a blue crossed tick indicates a partially supported feature. For obtaining the evaluation metrics, packages `survival`, `riskRegression`, `SurvMetrics` and `pec` require the fitted model object as input (a specific object format) and `RisksetROC` imposes to use a smoothing method. Packages `timeROC`, `riskRegression` and `pec` force the user to choose a form for subject-specific weights (e.g., inverse probability of censoring weighting (IPCW)). Packages `survcomp` and `SurvivalROC` do not implement the general AUC but the censoring-adjusted AUC estimator proposed by @Heagerty2000.\label{tab:bibliography_R}](table_2.png)


# Functionality

## Loss functions

**Cox loss function** is defined as the negative of the Cox proportional hazards model's partial log-likelihood [@Cox1972]. The function requires the subject-specific log relative hazards and the survival response (i.e., event indicator and time-to-event or censoring).  In case of ties in the event times, the user can choose between the Breslow method [@Breslow1975] and the Efron method [@Efron1977] to approximate the Cox partial log-likelihood.
```python
from torchsurv.loss import cox

# PyTorch model outputs ONE log hazard per observation
my_model = MyPyTorchCoxModel()

for data in dataloader:
    x, event, time = data  # covariate, event indicator, time
    log_hzs = my_model(x)  # torch.Size([64, 1]), if batch size is 64 
    loss = cox.neg_partial_log_likelihood(log_hzs, event, time)
    loss.backward()  # native torch backend
```

**Weibull loss function** is defined as the negative of the Weibull AFT's log-likelihood [@Carroll2003]. The function requires the subject-specific log parameters of the Weibull distribution (i.e., the log scale and the log shape) and the survival response. The log parameters of the Weibull distribution should be obtained from a `PyTorch`-based model pre-specified by the user.

```python
from torchsurv.loss import weibull

# PyTorch model outputs TWO Weibull parameters per observation
my_model = MyPyTorcWeibullhModel() 

for data in dataloader:
    x, event, time = data
    log_params = my_model(x)  # torch.Size([64, 2]), if batch size is 64 
    loss = weibull.neg_log_likelihood(log_params, event, time)
    loss.backward()
```

**Momentum** helps training model when the batch size is greatly limited by computational resources (i.e. large files). This impacts the stability of model optimization, especially when rank-based loss is used. Inspired from MoCO [@he2020momentum], we implemented a momentum loss that decouples batch size from survival loss, increasing the effective batch size and allowing robust train of a model, even when using a very limited batch size (e.g., $\leq 16$).

```python
from torchsurv.loss import Momentum
my_model = MyPyTorchXCoxModel()  
my_loss = cox.neg_partial_log_likelihood  # Works with any TorchSurv loss
momentum = Momentum(backbone=my_model, loss=my_loss)

for data in dataloader:
    x, event, time = data
    loss = model_momentum(x, event, time)  # torch.Size([16, 1])
    loss.backward()

# Inference is computed with target network (k)
log_hzs = model_momentum.infer(x)  # torch.Size([16, 1])
```

## Evaluation Metrics Functions

The `TorchSurv` package offers a comprehensive set of metrics to evaluate the predictive performance of survival models, including the `AUC`, `C-index`, and `Brier score`. The inputs of the evaluation metrics functions are the subject-specific risk score estimated on the test set and the survival response on the test set. The risk score measures the risk (or a proxy thereof) that a subject has an event.  We provide definitions for each metric and demonstrate their use through illustrative code snippets.

**AUC.** The AUC measures the discriminatory capacity of the survival model at a given time $t$, i.e., the ability to provide a reliable ranking of times-to-event based on estimated subject-specific risk scores [@Heagerty2005;@Uno2007;@Blanche2013].

```python
from torchsurv.metrics import Auc
auc = Auc()
auc(log_hzs, event, time)  # AUC at each time 
auc(log_hzs, event, time, new_time=torch.tensor(10.))  # AUC at time 10
```

**C-index.** The C-index is a generalization of the AUC that represents the assessment of the discriminatory capacity of the survival model across the entire time period [@Harrell1996;@Uno_2011].

```python
from torchsurv.metrics import ConcordanceIndex
cindex = ConcordanceIndex()
cindex(log_hzs, event, time)
```

**Brier Score.** The Brier score evaluates the accuracy of a model at a given time $t$ [@Graf_1999]. It represents the average squared distance between the observed survival status and the predicted survival probability. The Brier score cannot be obtained for the Cox proportional hazards model because the survival function is not available, but it can be obtained for the Weibull ATF model.

```python
from torchsurv.metrics import Brier
surv = survival_function(log_params, time) 
brier = Brier()
brier(surv, event, time)  # Brier score at each time
brier.integral()  # Integrated Brier score over time
```

**Additional features.** In `TorchSurv`, the evaluation metrics can be obtained for time-dependent and time-independent risk scores (e.g., for proportional and non-proportional hazards). Additionally, subjects can be optionally weighted (e.g., by the inverse probability of censoring weighting (IPCW)). Lastly, functionalities including the confidence interval, one-sample hypothesis test to determine whether the metric is better than that of a random predictor, and two-sample hypothesis test to compare two evaluation metrics between  different models are implemented. The following code snippet exemplifies the aforementioned functionalities for the C-index.

```python
cindex.confidence_interval()  # CI, default alpha=.05
cindex.p_value(alternative='greater')  # pvalue, H0:c=0.5, HA:c>0.5
cindex.compare(cindex_other)  # pvalue, H0:c1=c2, HA:c1>c2
```

# Conflicts of interest

MM, PK, DO and TC are employees and stockholders of Novartis, a global pharmaceutical company.

# References