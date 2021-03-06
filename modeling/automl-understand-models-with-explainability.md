# Understanding automated machine learning generated models, using the model explainability capability of automated machine learning

While training a good machine learning model is very important, it is almost equally important in many cases to have the ability to interpret and explain the behavior of that trained model. This capability is commonly referred as explainability or interpretability (sometimes these terms are used interchangeably). The insights it generates can be used to validate model behavior and various hypotheses, get insights for debugging, and, ultimately, build trust with the beneficiaries of the model.

There are two major aspects of model explainability:

- Understand global behavior (commonly referred as global explanation)
- Understand specific predictions generated by the model (commonly referred as local explanation)

## Explainability defined

The typical approach in machine learning is to use a set of data fields (referred as features) to predict the likely value of a target field. A trained model is essentially a function that takes the values of these features and computes a result which is the predicted value. One of the difficult problems with this approach has always been the opacity of this function or, in other words, the degree to which the inner working of the function can be understood and explained. This opacity depends quite a lot on the class of algorithm used to train the model. For example, decision tree algorithms produce one of the least opaque trained models (commonly referred as decision trees) which are essentially self-explanatory. At the other end of the spectrum, deep neural network algorithms produce of the opaquest models (which are essentially a set of numeric weights quite difficult to understand and explain).

One of the most important aspects of model explainability is **feature importance** which tells for each input feature how important is its contribution to the resulting prediction. Take for example the [bike sharing prediction](../automl-for-classification-regression-forecasting/automl-regression-code-sample.md) problem we have referred in a few places in this guide. Some of the features taken into consideration when predicting the number of bike rentals are `season`,`mnth`,`hr`,`holiday`,`weekday`,`workingday`,`weathersit`,`temp`,`atemp`,`hum`, and `windspeed`. Once a model is trained, one might find out that `hr` may be highly important in the final prediction while `weekday` may be less important.

The methods used to explain models can be also catgorized into:

- Model agnostic methods - can be applied to any model, regardless of the algorithm used to train it
- Model specifc methods - target only certain types of models (like tree-based models for example)

The following table lists the SDK packages that provide support for explainability.

| Package                              | Description                                                 |
| ------------------------------------ | ----------------------------------------------------------- |
| azureml.explain.model                | The main package, fully supported by Microsoft.             |
| azureml.train.automl.automlexplainer | The package for autoML models, also supported by Microsoft. |
| azureml.contrib.explain.model        | Experimental functionalities in preview, no support.        |

The SDK contains two sets of explainers (Direct Explainers and Meta Explainers) which are detailed below.

## Direct Explainers

Direct explainers are integrated into the SDK and expose a common output format and API. Thus, they can be directly invoked by your code. The following table lists the direct explainers available in the SDK.

| Explainer                                | Description                                                                                                                                                                                                        |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| SHAP Tree Explainer                      | Focused on trees and ensembles of trees (uses the polynomial time fast SHAP value estimation algorithm).                                                                                                           |
| SHAP Deep Explainer                      | Focused on deep learning models (uses a high-speed aproximation algorithm based on [DeepLIFT](https://arxiv.org/abs/1704.02685)). Has full support for TensorFlow (and Keras) and preliminary support for PyTorch. |
| SHAP Kernel Explainer                    | Can be applied to any model (uses a variant of linear regression to estimate SHAP values).                                                                                                                         |
| Mimic Explainer                          | Based on the concept of a global surrogate model that is intrinsically explainable. This surrogate model is trained to approximate the predictions of the original black box model.                                |
| Permutation Feature Importance Explainer | Focused on classification and regression models (uses an [approach proposed for random forests](https://www.stat.berkeley.edu/~breiman/randomforest2001.pdf)).                                                     |
| LIME Explainer                           | Focused on classifiers, aims to explain indifidual predictions by training local surrogate models.                                                                                                                 |
| HAN Text Explainer                       | Focused on text data (trains a surrogate model on a given model's outputs then optimizes for a specific document).                                                                                                 |

**Notes:**

The official defintion of [SHAP](https://shap.readthedocs.io/en/latest/) is:

> _SHAP (SHapley Additive exPlanations) is a unified approach to explain the output of any machine learning model. SHAP connects game theory with local explanations, uniting several previous methods and representing the only possible consistent and locally accurate additive feature attribution method based on expectations (see the SHAP NIPS paper for details)._

LIME stands for Local Interpretable Model-agnostics Explanations and has been originally proposed in the ["Why Should I Trust You?" - Explaining the Predictions of Any Classifier](https://arxiv.org/pdf/1602.04938.pdf) paper.

HAN stands for Hierarchical Attention Networks ahd has been originally proposed in the [Hierarchical Attention Networks for Document Classification](https://www.cs.cmu.edu/~diyiy/docs/naacl16.pdf) paper.

## Meta Explainers

As their name implies, Meta explainers are used for automatic selection of direct explainers. Based on a given models and dataset, a meta explainer will select the best direct explainer and use it to generate the explanation information. The following table lists the meta explainers available in the SDK.

| Explainer         | Description                |
| ----------------- | -------------------------- |
| Tabular Explainer | Used for tabular datasets. |
| Text Explainer    | Used for test datasets.    |
| Image Explainer   | Used for image datasets.   |

The following diagram shows the current relationship between meta and direct explainers.

![Meta and Direct Explainers in Azure Machine Learning service SDK](./media/automl-explainability-architecture.png)

## Model explainability code sample

### Pre-requisites

This code sample uses the results of the [Classification AutoML code sample](../automl-for-classification-regression-forecasting/automl-classification-code-sample.md). Make sure you have in your Azure Machine Learning service workspace at least one successful experiment run that yielded a trained classification model.

Note: All code snippets in this section are designed to run in [Azure Notebooks](https://notebooks.azure.com/).

### Prepare your environment

he first step you will perform is installing/updating your Azure Machine Learning service SDK:

```python
!pip install --upgrade azureml-sdk[notebooks,explain,automl,contrib]
```

Once you have the latest version of the SDK, you will need to do the necessary imports and specify the identifier of the experiment run:

```python
import pandas as pd

from azureml.train.automl.run import AutoMLRun

from azureml.train.automl.automlexplainer import explain_model
from azureml.train.automl.automlexplainer import retrieve_model_explanation

experiment_name = 'sonar-binary-classifier'
run_id = 'AutoML_3629f91f-628e-4f84-af18-862bd66287e5'
```

Note: You need to specify the experiment run identifier only if you are trying to get model explainability separately from executing the run.

### Prepare input data

Once everything is in place, you will load your input data and split it into the training and testing data sets:

```python
data_flow = dprep.read_csv('https://quickstartsws9073123377.blob.core.windows.net/azureml-blobstore-0d1c4218-a5f9-418b-bf55-902b65277b85/sonar/sonar.all-data.csv',
                           header=dprep.api.dataflow.PromoteHeadersMode.NONE,
                          infer_column_types=True)

X = data_flow.keep_columns(['Column{:d}'.format(x) for x in range(1,61)]).to_pandas_dataframe()
y = data_flow.keep_columns(['Column61']).to_pandas_dataframe()

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=111)
```

### Load the run

Next, you will get access to the run and retrieve the best model that was trained during execution:

```python
experiment = Experiment(ws, experiment_name)
automl_run = AutoMLRun(experiment, run_id)
best_run, fitted_model = automl_run.get_output()
```

Note: The creation of the `Workspace` variable `ws` is ommited for brewity.

### Explain your model (numerically)

Use the `retrieve_model_explanation` method to get the feature importance data for the best run.

![Feature importantce in AutoML - numeric](./media/automl-explainability-numeric.png)

Notice how the code assembles the global results into an easy to understand Pandas dataframe.

Since the predicted feature has two classes, the `per_class_imp` and `per_class_summary` variables will have a length of two, containing feature importance for each class. You can use the following code to retrieve feature importance for the first class:

```python
features_df = pd.DataFrame(list(zip(per_class_imp[0], per_class_summary[0])), dtype=float)
pd.options.display.float_format = '{:.5g}'.format
features_df.head()
```

### Explain your model (visually)

Use the `RunDetails` object to display the visual widget related to your run:

```python
from azureml.widgets import RunDetails
RunDetails(automl_run).show()
```

This will display the following dashboard:

![Feature importance in AutoML - details dashboard](./media/automl-explainability-visual-1.png)

Clicking on the top iteration (the best one) will enable you to view the graphical interpretation of the feature importance data:

![Feature importance in AutoML -visual](./media/automl-explainability-visual-2.png)

## Next steps

You can learn more about understanding automated machine learning generated models using the model explainability capability of azure machine learning by reviewing these links to additional resources:

- [What is automated machine learning?](https://docs.microsoft.com/azure/machine-learning/service/concept-automated-ml)
- [Model interpretability with Azure Machine Learning service](https://docs.microsoft.com/azure/machine-learning/service/machine-learning-interpretability-explainability)
- [Azure Machine Learning interpretability sample notebooks](https://github.com/Azure/MachineLearningNotebooks/tree/master/how-to-use-azureml/explain-model)

Read next: [Model deployment](../model-deployment/README.md)
