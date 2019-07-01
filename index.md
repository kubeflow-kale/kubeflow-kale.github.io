![Kale Banner](https://raw.githubusercontent.com/kubeflow-kale/kubeflow-kale.github.io/master/assets/imgs/banner.png)

---------------------------------------------------------------------

#### TL;DR

Kale (Kubeflow Automated Pipelines Engine) is a tool composed of a Jupyter extension and a Python engine that lets a user deploy a Jupyter Notebook into a [Kubeflow Pipelines](https://github.com/kubeflow/pipelines) workflow just by tagging the Notebook cells - no KFP SDK code required.

#### Introduction

In our experience, the machine learning and data processing prototyping phase often happens using Jupyter Notebooks. This can result in disorganized code that performs data processing, feature engineering, model definition, training and testing - all into the same notebook. Experiments can become hard to maintain and reproduce, possibly leading to wrong results and a lot of additional work. A messy ML pipeline makes it much harder also to deploy it in the Cloud for scale up and production.

Kubeflow is becoming the standard de-facto platform when dealing with large scale machine learning workflows on top of Kubernetes. It is designed to simplify the setup, deployment and monitoring of Machine Learning jobs both in the Cloud and on-premises clusters, leveraging on and ecosystem of Cloud Native components. Together with the newly introduced Kubeflow Pipelines (KFP) platform, Kubeflow can help data scientist and software engineers to better organize end-to-end Machine Learning projects, from on-premise prototyping to scaling up in the Cloud and production serving.

---

Despite being designed to be simple and accessible, Kubeflow Pipelines' Python SDK can still result difficult for ML practitioners without a software engineering background. Many data scientists, especially in the research field, come from a physics or maths backgrounds, often lacking the engineering expertise to develop complex deployment and infrastructure setup procedures.

Converting an existing Jupyter Notebook - running on-prem, even on a laptop - to a KFP pipeline, can become a challenging task when dealing with complex workflows. 


#### Kale: a simpler KFP interface designed for Data Scientists

Kale was designed to address the difficulties by providing a tool to simplify the deployment process of a Jupyter Notebook into Kubeflow Pipelines workflows. Translating Jupyter Notebook directly into a KFP pipelines ensure that all the processing building blocks are well organized and independent from each other, while also leveraging on the experiment tracking and organization provided out-of-the-box by Kubeflow.

The main idea behind Kale is to exploit the built-in Jupyter tagging feature [^1] [^2] to:

1. assign cells to specific pipeline components
2. define the (execution) dependencies between them
3. merge together multiple cells into a single pipeline component

Kale takes as input the tagged Jupyter Notebook and automatically generates a standalone Python script that describes the KFP pipeline using a series of KFP *lightweight components*, corresponding to the tagged jupyter cells.

*Image here showing a few tagged cells and the resulting pipeline*

One question you might raise is: *How does Kale manage to resolve the data dependencies between the steps, thus making available the notebook variables throughout the pipeline execution?*

Kale runs a series of static analyses over the Python components to detect where variables and objects are first declared and then used. In this way Kale creates an internal graph representation describing the data dependencies between the steps. Using this knowledge, Kale marshals these objects between the pipeline steps by serializing the variables into a shared PVC. Both marshalling and management of the shared PVC is done transparently to the user, who does not need to worry about this.








Kale is a Python package that aims at automatically deploy a general purpose Jupyter Notebook as a running [Kubeflow Pipelines](https://github.com/kubeflow/pipelines) instance, without requiring the use the specific KFP DSL.

The general idea of kale is to automatically arrange the cells included in a notebook, and transform them into a unified KFP-compliant pipeline. To do so, the user is only required to decide which cells correspond to which pipeline step, by the use of tags. In this way, a researcher can better focus on building and testing its code locally, and then scale it in a simple, organized and controlled way.

## Tagging language

Jupyter provides a tagging feature out-of-the-box, that lets you associate each cells with custom defined tags. The feature is available also in JupyterLab via the [jupyterlab-celltags](https://github.com/jupyterlab/jupyterlab-celltags) extension.

The tags are used to tell Kale how to convert the notebook's code cells into an execution graph, by specifying the execution dependencies between the pipeline steps and which code cells to merge together.

This is a list of tags recognized by Kale:

| Tag | Description | Example |
| :---: | :---: | :---: |
| `^block:<block_name>(;<block_name>)*$` | Assign the current cell to a (multiple) pipeline step | `block:train-model`<br>`block:processing-A;processing-B`|  
| `^prev:<block_name>(;<block_name>)*$` | Define an execution dependency of the current cell to `n` other pipeline steps | `prev:load-dataset
| <code>imports&#124;functions</code> | Tell Kale to add this code block at the beginning of every pipeline code block. Useful to add imports/function to every pipeline step | - |  
| `skip` | 'Hide' the current cell from Kale. | - |

Where `<block_name>` is matched against the regex: `[a-z0-9]`. So any string containing only digits and lowercase characters.

## Installation

Kale is provided as a Python package. Just clone the repository to your local machine and install the package in a virtual environment. 

```bash
# Clone the repo to your local environment
git clone https://github.com/StefanoFioravanzo/kale.git
cd kale
# Install the package in your virtualenv
python setup.py install
```

## Getting Started

First you need to have a running Kubeflow instance (Kubeflow [getting started guide](https://www.kubeflow.org/docs/started/getting-started/)).


Kale provides a CLI command. Run `kale --help` for a detailed description of the execution parameters.

Example:

```bash
kale --nb examples/base_example_numpy.ipynb \
	--pipeline_name numpy_example \
	--pipeline_descr "Numpy Example" \
	--docker_image stefanofioravanzo/kale-kfp-examples:0.1
```
This will produce a python script `examples/kfp_numpy_examples.kfp.py` containing all the definitions of the KFP stand-alone functions and the necessary code to define and deploy a pipeline. If run with the `--deploy` flag, Kale will try to automatically deploy the generated pipeline (default KFP url is `localhost:8080`, see `kale` CLI parameters for customization).

Have a look at the notebooks under the examples folder to start experimenting with Kale.

## Architecture

`core.py` contains the `Kale` class, main entry-point of the application. The function `run` does the conversion from jupyter notebook to kfp pipeline by calling all the other sub-modules.

Kale was developed with a modular design. There are 4 main modules, each with a specific functionality.

#### 1. nbparser

This module both defines the Kale's Jupyter tagging language and provides the machinery to convert a tagged notebook to a NetworkX graph representing the resulting KFP pipeline. The main function is `parse_notebook()`, which takes as input a tagged notebook and returns the graph. The pipeline steps Python code is contained inside a special attributed of the graph's nodes.

#### 2. static_analysis

The purpose of this module is to detect the data dependencies between the code snippets in the execution graph. This is achieved by running PyFlakes that is able to detect all the potential missing variables errors, plus some custom routines that iterate over the AST. `dep_analysis.variables_dependencies_detection()` is the main function called by Kale to start the static analysis process.

#### 3. marshal

Marshal is a module that implements two dispater classes. `PatternDispatcher` can register functions based a regex pattern, and then call one of these functions based on a user provided string. `TypeDispatcher` can register functions based on a pattern as well, but dispatches a function based on the *type* of the input objects instead of a string.

Example:

```python
from kale.marshal import resource_save, resource_load

# register functions to load and save numpy objects
@resource_load.register('.*\.npy')  # match anything ending in '.npy'
def resource_numpy_load(uri, **kwargs):
    return np.load(uri)

@resource_save.register('numpy\..*')  # match anything starting with 'numpy'
def resource_numpy_save(obj, path, **kwargs):
    return np.save(path+".npy", obj)
    
a = np.random.random((10,10))
resource_save(a, 'marshal_dir/')
b = resource_load('marshal_dir/a.npy')  # a == b
```

Common backends to support multiple data types are registered in `marshal/backends.py`. The idea is to provide backends for the most used data types and have a common fallback to standard `dill` serialization in case the data type is not recognized.

**NB:** The code in this module is not used directly by Kale during the conversion process from notebook to pipeline, but is instead injected at the beginning and at the end of each generated pipeline step to serialize and de-serialize objects that need to be passed between pipeline steps. Have a look at `templates/function_template.txt` and at the generated python script (e.g. `kfp_numpy_example.kfp.py`).

#### 4. codegen

This module provides a single function `gen_kfp_code` that exploits the Jinja2 templating engine to generate a fully functional python script. The templates used are under the `templates/` folder.

## Flask Server

The package provides the `kale_server` CLI command that runs a Flask server. The Flask server accepts requests at `localhost:5000/kale`. The API accepts POST requests containing a Jupyter Notebook in raw format (JSON) and will call the main Kale module to deploy the notebook to a KFP instance.

`kale/` API available parameters:

- `nb`: Raw JSON Jupyter Notebook
- `deploy`: Boolean Flag. `True` to deploy the Notebook automatically
- `kfp_port`: Port of the running KFP instance
- `pipeline_name`: Name of the resulting KFP pipeline
- `pipeline_descr`: Description of the resulting KFP pipeline
- `docker_image`: Docker image to use for the pipeline steps

## JupyterLab Extension

Under `extensions/kale-toolbar-runner` you can find the code of a JupyterLab extension that adds a new Button in the Notebook toolbar. The related action will call the `localhost:5000/kale` API sending the currently active Notebook. This way one can easily deploy a Notebook to KFP without leaving the JupyterLab UI.

