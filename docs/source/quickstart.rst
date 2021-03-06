Quickstart
==========
To use maxai, first install it using pip

::

  (.venv) $ pip install maxai-1.0.0-py3-none-any.whl


maxairesources
==============
maxairesources library consists of various helper methods which includes utilities to perfrom data quality checks, build spark pipelines, logging, model approval etc. Here is the detailed list of helper methods:


Datachecks
___________


*DataChecks* module lets you perform various checks on data including univariate summary stats such as mean, media, mode, kurtosis, skewness, null values and bivariate correlation analysis

- It consists of `AnalysisBase` class is a base class in `analysis_base.py` file which should be inherited by all analysis class specific to use case / requirement.
- Every analysis class will produce dictionary, which can be saved into disk using `save_analysis_report`
- Types of columns are identified based on column names as per feast output, or you can pass dictionary as shown in below code snippet
- `SparkDataFrameAnalyser` class is used to analyse `pyspark` `dataframe` with numerical and categorical columns mainly

::

  from maxairesources.datachecks.dataframe_analysis_spark import SparkDataFrameAnalyser
  
  col_types = {
    "numerical_cols": [], # numerical columns
    "bool_cols": [],# boolean columns with True / False values
    "categorical_cols": [], # with categorical columns only, not text columns
    "free_text_cols": [], # text column (NLP) TODO 
    "unique_identifier_cols": [] # unique ID columns (primary key)
  }
  df = "<your pyspark dataframe>" # your data here
  analyser = SparkDataFrameAnalyser(df=df, column_types=col_types) # create instance of analyser
  report = analyser.generate_data_health_report() # generate report
  analyser.save_analysis_report(report) # saves in json file
  
Note: while running `generate_data_health_report` method, report will be prepared and all data health calculations / checkup results will be printed in two logging levels
  1. logging.WARNING: shows that data is not aligned with given thresholds, appropriate transformation might required on dataset
  2. logging.INFO: shows information about specific checkup, no action required.

We can also provide `threshold` parameter value for checkups in following format
::

  thresholds = {
    "general": {"column_count": 5, "record_count": 10},
    "uni_variate": {
        "skewness_score": 1.0,
        "kurtosis_score": 3,
        "outlier_percentage": 1,
        "null_percentage": 1,
        "unique_categories_count": 10,
        "stddev": 3,
        "boolean_balance": 0.1,
    },
    "bi_variate": {"correlation_score": 0.5},
  }
  

Preprocessing
___________

Preprocessing is a class that covers the basic pre-processing functions to create aggregated features from transaction data to a customer level. It can be used to convert the transactional data to cross-sectional data.

Here is an example:

::

  ## Load requirements
  # Load libraries
  import pyspark
  from pzai.pzaiutils.datamanager import DataFrame
  from pzaipreprocessing.preprocessing import Preprocessing

  # Define input_data_json and input_port_number

  # Read data in pyspark
  data = DataFrame().get(input_data_json, port_number=input_port_number)

Operations

::

  # The following operations are currently supported by the preprocessing module.
  1: "sum",
  2: "mean", 
  3: "median", 
  4: "max", 
  5: "min", 
  6: "count", 
  7: "countDistinct", 
  8: "standard_deviation"

Examples of using PreProcessing artifact

::

  # Example 1: To find the distinct number of products for each product category for all the customers in the last 1 year
  # event_column = "product_category"
  # agg_col = "product_name"
  # time_period = 365

  output_df = Preprocessing().transaction_cross_section(dataframe=df, groupby_col = "cust_id", arguments = [{"event_column": "product_category", "agg_col": "product_name", "filter": "all", "operation": 7, "time_period": 365}])

  # Example 2: To find the sum of revenue for specific product name from the product category for all the customers in the last 100 days
  # event_column = "product_category"
  # agg_col = "product_revenue"
  # time_period = 100

  output_df = Preprocessing().transaction_cross_section(dataframe=df, groupby_col = "cust_id", arguments = [{"event_column": "product_category", "agg_col": "product_revenue", "filter": "ABC", "operation": 1, "time_period": 100}])

  # Example 3: To find the standard deviation of revenue for product category for all the customers in the last 200 days
  # event_column = "product_category"
  # agg_col = "product_category"
  # time_period = 200

  output_df = Preprocessing().transaction_cross_section(dataframe=df, groupby_col = "cust_id", arguments = [{"event_column": "product_category", "agg_col": "product_category", "filter": None, "operation": 8, "time_period": 200}])

So, here, we have filter = None, all, specific element, and time period can be varied according to convenience, 8 different aggregation can be performed, event 
column and agg_column can be used as required.


SparkPipeline
______________
**'SparkPipline'** offers an abstraction over transformers and estimator pipelines in PySpark, Here is how you can use this utility in your workflow.

::
  

  from maxairesources.pipeline.spark_pipeline import SparkPipeline

  # input training dataframe
  training = spark.createDataFrame([
          (0, "a b c d e spark", 1.0),
          (1, "b d", 0.0),
          (2, "spark f g h", 1.0),
          (3, "hadoop mapreduce", 0.0)
      ], ["id", "text", "label"])

  # input test or scoring dataframe
  test = spark.createDataFrame([
      (4, "spark i j k"),
      (5, "l m n"),
      (6, "spark hadoop spark"),
      (7, "apache hadoop")
  ], ["id", "text"])

  # create a sparkpipline of transformers/estimators and their arguments as key value pairs as shown below
  sp = SparkPipeline({'Tokenizer':{'inputCol':'text','outputCol':'words'},
   'HashingTF':{'inputCol':'words','outputCol':'features','numFeatures':1024}})

  # fit a pipeline on training data
  sp.fit_pipeline(training)

  # call transform_pipeline on fitted pipeline to transform test data
  sp.transform_pipeline(test)



  # create a sparkpipline of same set of transformers/estimators and their arguments as key value pairs for multiple columns 
  # with same pipeline
  # Example:

  # input training dataframe
  training = spark.createDataFrame([
          (0, "a b c d e spark", "machine learning", 1.0),
          (1, "b d","deep learning", 0.0),
          (2, "spark f g h", "natural language processing",1.0),
          (3, "hadoop mapreduce","computer vision", 0.0)
      ], ["id", "text","domains", "label"])

  # input test or scoring dataframe
  test = spark.createDataFrame([
      (4, "spark i j k", "machine"),
      (5, "l m n", "learning"),
      (6, "spark hadoop spark", "language"),
      (7, "apache hadoop", "vision")
  ], ["id", "text", "domains"])


  # if you have to apply the same transformations for two text columns 
  # consider below as an example. Below is the dictionary created for two text columns.
   {'Tokenizer': {'inputCol': 'text', 'outputCol': 'texttk'},
    'StopWordsRemover': {'inputCol': 'texttk', 'outputCol': 'textsw'},
    'HashingTF': {'inputCol': 'textsw','outputCol': 'texthtf','numFeatures': 1024},
    'IDF': {'inputCol': 'texthtf', 'outputCol': 'textidf'},
    'Tokenizer': {'inputCol': 'domains', 'outputCol': 'domainstk'},
    'StopWordsRemover': {'inputCol': 'domainstk', 'outputCol': 'domainssw'},
    'HashingTF': {'inputCol': 'domainssw','outputCol': 'domainshtf','numFeatures': 1024},
    'IDF': {'inputCol': 'domainshtf', 'outputCol': 'domainsidf'},
    'VectorAssembler': {'inputCol': ['textidf', 'domainsidf'],'outputCol': 'assembler_features'},
    'MinMaxScaler': {'inputCol': 'assembler_features','outputCol': 'scaled_features'}}


  text_cols = ['text','domains']
  cols = []
  transformation_dict = {}
  for i in text_cols:
      transformation_dict[i] = {'Tokenizer':{'inputCol':i,'outputCol':i+'tk'},
       'StopWordsRemover':{'inputCol':i+'tk','outputCol':i+'sw'},
       'HashingTF':{'inputCol':i+'sw','outputCol':i+'htf','numFeatures':1024},
       'IDF': {'inputCol':i+'htf','outputCol':i+'idf'}}
      cols.append(i+'idf')

  transformation_dict['vectorassembler'] = {'VectorAssembler': {'inputCols': ['textidf','domainsidf'], 'outputCol':"assembler_features"}}
  transformation_dict['MinMaxScaler'] = {'MinMaxScaler' : {'inputCol': 'assembler_features', 'outputCol':"scaled_features"}}
  transformation_dict

  sp = SparkPipeline(transformation_dict)
  sp.fit_pipeline_multiple(training)
  sp.transform_pipeline(retail_dcf_temp_label)


Logging
_______

Generic logging module available in max to log objects in a workflow The logging method is in `maxairesources/logging/logger.py` file. use `get_logger` method to get logger object.

::

  from maxairesources.logging.logger import get_logger
  logger = get_logger(__name__)
  

logger support 5 levels of logging as below.

::

  | Level      | When it's used                                                                                                                                                                                                                                                                                                                                                                                                                                           |
  |------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | `DEBUG`    | Detailed information, typically of interest only when diagnosing problems. <br />Example<br />- Can be used to print intermediate information to debug code blocks <br />- Number of observations, column list in `Spark` `Dataframe` <br />- Parameters received to train the model<br />- `train` and `test` data size<br /><br />Do not print any raw data / information in debug messages as some data may be confidential to display in `log` also. |
  | `INFO`     | Confirmation that things are working as expected. <br />Example<br />- Log success message once model is trained<br />- Inform that `model` is persisted in disk space                                                                                                                                                                                                                                                                                   |
  | `WARNING`  | An indication that something unexpected happened, or indicative of some problem in the near future (e.g. ???disk space low???). The software is still working as expected.<br />Example<br />- Warn user if data size is less<br />- Highlight long processing time if model parameters grid combination for optimization are more than limit.                                                                                                               |
  | `ERROR`    | Due to a more serious problem, the software has not been able to perform some function.<br />Example<br />- If `data frame` is empty when observations are expected<br />- Fail fast model checks are not passing                                                                                                                                                                                                                                        |
  | `CRITICAL` | A serious error, indicating that the program itself may be unable to continue running.<br />Example<br />- Database credentials are incorrect<br />- Certain path is not accessible from current user                                                                                                                                                                                                                                                    |

- Currently, logger support two types of handlers

1. `FileHandler`: produce log file which could be viewed using text editor and 
2. `StreamHandler`: send log messages to `terminal` `console`. This also gets printed along with spark log

- Log format

  ```
  %(asctime)s - [ID:xxx] [%(levelname)s] - [(%(name)s) - (%(filename)s) - (%(funcName)s) - line %(lineno)d]- [%(message)s]
  ```

- Example of usage

  from maxairesources.logging.logger import get_logger #import function
  logger = get_logger(__name__) #get logger
  logger.debug(f"log this debug message") #log debug message
 
 
Multi Train
______________

**Multi Train** class lets you train multiple models in parallel. 
Here is a working example

::

  from maxairesources.utilities.multi_train import MultiTrain
  models = {
          "SparkGBTClassifier": {
              "id_col": None,
              "target_col": "label",
              "feature_col": "features",
              "params": {"maxIter": 3, "maxDepth": 3, "seed": 42},
              "param_grid": {},
          },
          "SparkRFClassifier": {
              "id_col": None,
              "target_col": "label",
              "feature_col": "features",
              "params": {"maxDepth": 3, "seed": 42},
              "param_grid": {},
          },
      }
  multi_models = MultiTrain(models)


Ensemble
______________

**Ensemble** class lets you create an ensemble of multiple models. The class supports following ensemble techniques

  1. **Voting Classifier**: Consists of three ensemble methods - hard, soft, weighted soft
            **Hard Voting** - We will calculate the mode of prediction across all the classifiers, and provide the Combined Prediction label as the output
            **Soft Voting** - Here if the user doesn't enter the weights, we will calculate the uniform average of probabilities across all the classifier outputs, and 
            return Average Probability Column as the output.
            
            **Weighted Soft Voting** - Here if the user enter the weights, we will calculate the weighted average of probabilities across all the classifier outputs,
            and return weighted Average Probability Column as the output. 

  2. **VotingRegressor** - Consists of two ensemble methods - soft, weighted soft
            **Soft Voting** - Here if the user doesn't enter the weights, we will calculate the uniform average of predictions across all the regressor outputs, and               return Average Prediction Column as the output.
            
            **Weighted Soft Voting** - Here if the user enter the weights, we will calculate the weighted average of predictions across all the regressor outputs, and              return weighted Average Prediction Column as the output.

Here is a working example

::

  from maxairesources.ensemble.ensemble import Ensemble
  model_list = []
  for i in range(len(multi_models.trained_models)):
      model_list.append(multi_models.trained_models[list(multi_models.models.keys())[i]])
  print(model_list)
  prediction = Ensemble(model_list).VotingClassifier(testData, method = "hard")

Model approval
______________

**`ModelApprover`** class checks whether the model performance is good enough based on existing benchmarks

`Approver` class needs `Evaluator` class reference along with other arguments in constructors. All required `constructor argument` for respective `evaluator` needs to pass as a `keyword argument` . Please refer `evaluator` documentation for details.

Here is a working example

::

  from maxairesources.eval.classifier_evaluator_spark import ClassifierEvaluator
  from maxairesources.model_approval.model_approver_spark import ModelApprover
  
  apprvr = ModelApprover(
    model = model,
    evaluator_class=ClassifierEvaluator,
    predicted_actual_pdf = predicted_pdf,
    metric_thresholds={"f1": 0.4, "accuracy": 0.55},
    predicted_col="prediction",
    label_col="label",
    probability_col="probability",
    classification_mode = "binary"
    )

Config Store
____________

**Config Store** lets you efficiently read secrets/configs in a task from a vault

The [HashiCorp's Vault](https://www.vaultproject.io/docs) is currently being used as a config store, to store the Py-Configs and Spark-Configs. The Vault provides the option to create a Secret Engine (represented by `mount_path` in code snippet below). All secrets are stored in a Secret Engine and can also have a directory structure. 

*Assumptions* - This module assumes that OS environment variables HASH_VAULT_URL and HASH_VAULT_TOKEN are defined. 

*Usage* - The `config_store.config.main` can be used to a function where one wants to read these secrets/configs. The best practise would be read these secrets/configs once in a task, because everytime we make a call to `config_store.config.main`, it creates a temporary token to read these secrets.

::

  #Example of Usage
  
  PATH = ""          # Path to the Config
  MOUNT_PATH = ""    # Secret Engin

  @config.main(path=PATH, mount_point=MOUNT_PATH)
  def execute(**kwargs):
      input_data = kwargs["data"]
      print("Printing Config = {}".format(input_data))

  #Printing Config = {'split_seed': 19, 'target_column': 'target', 'test_size': 0.2}

maxaifeaturization
==================
maxaifeaturization library has various helper methods to enable feature generation, feature selection and feature transformation. 

FeatureSelector
_______________

**'FeatureSelector'** offers an abstraction for selecting features using the methods available in pyspark feature selection, 
Class expects method to use for fearure selection and corresponding as inputs. 

Currently supported methods are
::

  selectors = {
          "VectorSlicer": {
              "model": VectorSlicer,
              "fitted_model": VectorSlicer,
              "type": "transform",
          },
          "RFormula": {"model": RFormula, "fitted_model": RFormula, "type": "transform"},
          "ChiSqSelector": {
              "model": ChiSqSelector,
              "fitted_model": ChiSqSelectorModel,
              "type": "fit",
          },
          "UnivariateFeatureSelector": {
              "model": UnivariateFeatureSelectorN,
              "fitted_model": UnivariateFeatureSelectorModel,
              "type": "fit",
          },
          "VarianceThresholdSelector": {
              "model": VarianceThresholdSelector,
              "fitted_model": VarianceThresholdSelectorModel,
              "type": "fit",
          },
      }

Here is how you can use this utility in your workflow.

::

  # import FeatureSelector from maxaifeaturization
  from maxaifeaturization.selection.selector import FeatureSelector

  # Initializing FeatureSelector class
  fs = FeatureSelector(method = 'UnivariateFeatureSelector', 
                       params = {'featuresCol':"features",
                        'outputCol':'selectedFeatures',
                        'labelCol':'label',
                        'selectionThreshold':1,
                        'featureType':'continuous',
                        'labelType':'categorical'})


  # select features using the passed method
  fs.select_features(feature_df)

  #access the underlying spark feature selection method object
  fs.selector

  # save the model
  fs.save('path')

  # load the model
  fs.load('path')

maxaimetadata
=============

Max AI Metadata 

*maxaimetadata* library offers classes and funtions to log ml-metadata for lineage tracking. Given below is a short description of various components within the library along with their functionality


**WorkFlow**

Collection of all the elements related to a datascience workflow. 

Workflow represent a jupyter notebook for a usecase or an airflow pipeline. 

if workflow already exists in the backend , it will get reused.



::

  # import WorkFlow from maxaimetadata
  from maxaimetadata.metadata import WorkFlow

  # Initializing WorkFLow class
  wf = WorkFlow(
          name="Propensity1",
          description="test workflow",
          tags={"sample": "sample"},
          reuse_workflow_if_exists=True,
      )

**Run**

Captures a particular instance/run of the worlflow. A workflow can have multiple runs.

::

  # import WorkFlow from maxaimetadata
  from maxaimetadata.metadata import Run

  # Initializing Run class
  run = Run(workflow=wf, description="test run")
  run.update_status("running")

**Execution**

Represent a task in the workflow [training, preprocessing , validation etc]

::

  from maxaimetadata.metadata import Execution

  exec = Execution(
          name="test exec", workflow=wf, run=run, description="test execution"
      )

**Artifacts**

Artifacts reperesents input/output of any execution. Eg: Model, Data , Metrics etc


::

  from maxaimetadata.metadata import Execution, Model, DataSet, Metrics

  d = DataSet(
          uri="/data", name="test_data", description="test data", feature_view="test_iew"
      )

  d = exec.log_input(d)

  #model is any MaxAi Model
  m = Model(model=model, name="test_model", description="test model")
  m = exec.log_output(m)

  metrics = Metrics(
      name="Test Metrics", data_set_id=d.id, model_id=m.id, values={"rmse": 0.9}
  )
  metrics = exec.log_output(metrics)


**Registry**
Model registry represent a logical collection of models registered for Inference.

::

  from maxaimetadata.metadata import Registry

  r = Registry(wf)
  r.register_model(m.uri)
  r_m = r.get_registered_model("staging")
  p_m = r.promote_model(r_m["__maxai_version__"])

Here is a sample output artifact from ml metadata
::

  [{'name': 'run_5ed07d01-5ed0-4663-a5fc-6cbadefa55ef',
    'id': 2,
    'create_time': 1654005916855,
    'repo': 'https://personalize-ai@dev.azure.com/personalize-ai/personalize.ai/_git/max.ai.ds.core',
    'branch': 'elastic-search',
    'commit': 'dac9a98744ab17963f806b313b18bf3be819d54e',
    'workflow': 'Propensity',
    'status': 'running',
    'description': 'Propensity sample run',
    'executions': [{'id': 1,
      'create_time': 1654006365178,
      'workflow': 'Propensity',
      'run': 'run_5ed07d01-5ed0-4663-a5fc-6cbadefa55ef',
      'tags': {'args': [], 'kwargs': {}},
      'name': 'model_train',
      'artifacts': [{'id': 1,
        'uri': 's3a://zs-sample-datasets-ds/credit-risk/customer',
        'create_time': 1654006365180,
        'workflow': 'Propensity',
        'run': 'run_5ed07d01-5ed0-4663-a5fc-6cbadefa55ef',
        'tags': None,
        'feature_view': 'customer',
        'name': 'customer',
        'type': 'maxai/DataSet'},
       {'id': 2,
        'uri': 's3a://zs-sample-datasets-ds/Propensity/run_5ed07d01-5ed0-4663-a5fc-6cbadefa55ef/Model/SparkGBTClassifier',
        'create_time': 1654006375920,
        'workflow': 'Propensity',
        'run': 'run_5ed07d01-5ed0-4663-a5fc-6cbadefa55ef',
        'tags': None,
        'name': 'SparkGBTClassifier',
        'training_framework': 'spark',
        'description': 'SparkGBTClassifier model',
        'hyperparameters': '{"params": {"maxIter": 5, "maxDepth": 3, "seed": 42}, "param_grid": {}}',
        'type': 'maxai/Model'}]}]}]


maxaimodel
==========

maxaimodel class support various Pyspark, Python and H2O models ranging from classification, clustering, regression to time-series forecasting. Here is a coprehensive list of models currently supported by max

1. Classification
  a) `SparkGBTClassifier <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.classification.GBTClassifier.html>`_
  b) `SparkRandomForestClassifier <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.classification.RandomForestClassifier.html#pyspark.ml.classification.RandomForestClassifier>`_
  c) `SparkFMClassifier <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.classification.FMClassifier.html#pyspark.ml.classification.FMClassifier>`_
  d) `SparkDecisionTreeClassifier <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.classification.DecisionTreeClassifier.html#pyspark.ml.classification.DecisionTreeClassifier>`_
  e) `SparkLogisticRegression <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.classification.LogisticRegression.html#pyspark.ml.classification.LogisticRegression>`_
  f) `SparkMultilayerPerceptronClassifier <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.classification.MultilayerPerceptronClassifier.html#pyspark.ml.classification.MultilayerPerceptronClassifier>`_
  g) `SparkNaiveBayes <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.classification.NaiveBayes.html#pyspark.ml.classification.NaiveBayes>`_
  h) `SparkOneVsRest <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.classification.OneVsRest.html#pyspark.ml.classification.OneVsRest>`_
  i) `SparkLinearSVC <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.classification.LinearSVC.html#pyspark.ml.classification.LinearSVC>`_
  
2. Clustering
  a) HVT
  b) `KMeans <https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.clustering.KMeans.html#pyspark.ml.clustering.KMeans>`_
  
3. Regression
  a) SparkDTRegressor
  b) SparkFMRegressor
  c) SparkGBTRegressor
  d) SparkGLRegressor
  e) SparkIsotonicRegressor
  f) SparkLinearRegressor
  g) SparkRFRegressor
  
4. Recommendation
  a) ALS
  
5. Forecasting
  a) ARIMA
  b) Garch
  c) NProphet
  d) FBProphet
  

tutorials
=========

Propensity to Churn
_______________

::

  from maxairesources.utilities.data_connectors import DataFrame
  from maxaimarketplace.high_tech.churn.handlers.churn_modeling import ChurnModeling
  
  #config load
  # Opening JSON file
  config_py_config = "/home/jovyan/new_max/max.ai.ds.core/maxaimarketplace/high_tech/churn/config/py_config.json"
  f = open(config_py_config)  
  py_config = json.load(f)
  py_config_feature_eng = py_config['data'][0]['conf']
  py_config_modeling = py_config['data'][1]['conf']
  py_config_scoring = py_config['data'][2]['conf']
  py_config_modeling
  
  #load data
  io_connector = DataFrame(spark_conn=spark)
  subscription_data_pdf = io_connector.get(input_data=py_config_feature_eng['input'], port_number=1)#.repartition(50)
  subscription_data_pdf.printSchema()
  
  #feature engineering and target variable
  from maxaimarketplace.high_tech.churn.handlers.churn_preprocessing import DataPreprocessing
  dp_obj = DataPreprocessing(input_data=subscription_data_pdf, args=py_config_feature_eng['function']['args'])
  training_data, scoring_data = dp_obj.preprocess_data()
  
  from maxaimarketplace.high_tech.churn.handlers.churn_modeling import ChurnModeling
  py_config_modeling['function']['args']['acceptance_thresholds']['f1'] = 0.25
  
  #training & validation
  model_instance = ChurnModeling(input_data=training_data, args=py_config_modeling['function']['args'], output_args=py_config_modeling['output'])
  model, performance_details = model_instance.train_and_validate_model()
  print(performance_details)
  # {'f1': 0.4783,
  #  'accuracy': 0.3208,
  #  'weightedPrecision': 0.9847,
  #  'weightedRecall': 0.3208,
  #  'weightedTruePositiveRate': 0.3208,
  #  'weightedFalsePositiveRate': 0.336,
  #  'weightedFMeasure': 0.4783,
  #  'logLoss': 1.0306,
  #  'hammingLoss': 0.6792}
  
  #scoring
  from maxaimarketplace.high_tech.churn.handlers.churn_scoring import ChurnScoring
  scoring_instance = ChurnScoring(scoring_data=scoring_data, args=py_config_scoring['function']['args'], input_args = py_config_scoring['input'], 
  output_args=py_config_scoring['output'])
  prediction_pdf = scoring_instance.predict_the_churn()
  
