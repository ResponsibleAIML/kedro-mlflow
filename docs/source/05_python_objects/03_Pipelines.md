# Pipelines
## ``PipelineML`` and ``pipeline_ml``

``PipelineML`` is a new class which extends ``Pipeline`` and enable to bind two pipelines (one of training, one of inference) together. This class comes with a ``KedroPipelineModel`` class for logging it in mlflow. A pipeline logged as a mlflow model can be served using ``mlflow models serve`` and ``mlflow models predict`` command.  

The ``PipelineML`` class is not intended to be used directly. A ``pipeline_ml`` factory is provided for user friendly interface.

Example within kedro template:

```python
# in src/PYTHON_PACKAGE/pipeline.py

from PYTHON_PACKAGE.pipelines import data_science as ds

def create_pipelines(**kwargs) -> Dict[str, Pipeline]:
    data_science_pipeline = ds.create_pipeline()
    training_pipeline = pipeline_ml(training=data_science_pipeline.only_nodes_with_tags("training"), # or whatever your logic is for filtering
                                    inference=data_science_pipeline.only_nodes_with_tags("inference"))

    return {
        "ds": data_science_pipeline,
        "training": training_pipeline,
        "__default__": data_engineering_pipeline + data_science_pipeline,
    }

```
Now each time you will run ``kedro run --pipeline=training`` (provided you registered ``MlflowPipelineHook`` in you ``run.py``), the full inference pipeline will be registered as a mlflow model (with all the outputs produced by training as artifacts : the machine learning, but also the *scaler*, *vectorizer*, *imputer*, or whatever object fitted on data you create in ``training`` and that is used in ``inference``).

*Note: If you want to log a ``PipelineML`` object in ``mlflow`` programatically, you can use the following code snippet:*

```python
from pathlib import Path
from kedro.framework.context import load_context
from kedro_mlflow.mlflow import KedroPipelineModel

# pipeline_training is your PipelineML object, created as previsously
catalog = load_context(".").io

# artifacts are all the inputs of the inference pipelines that are persisted in the catalog
pipeline_catalog = pipeline_training.extract_pipeline_catalog(catalog)
artifacts = {name: Path(dataset._filepath).resolve().as_uri()
                for name, dataset in pipeline_catalog._data_sets.items()
                if name != pipeline_training.model_input_name}


mlflow.pyfunc.log_model(artifact_path="model",
                        python_model=KedroPipelineModel(pipeline_ml=pipeline_training,
                                                        catalog=pipeline_catalog),
                        artifacts=artifacts,
                            conda_env={"python": "3.7.0"})
```