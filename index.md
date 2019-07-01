![Kale Banner](https://raw.githubusercontent.com/kubeflow-kale/kubeflow-kale.github.io/master/assets/imgs/banner.png)

---------------------------------------------------------------------

> #### TL;DR

> Kale (Kubeflow Automated Pipelines Engine) is a tool composed of a Jupyter extension and a Python engine that lets a user deploy a Jupyter Notebook into a [Kubeflow Pipelines](https://github.com/kubeflow/pipelines) workflow just by tagging the Notebook cells - no KFP SDK code required.

#### Introduction

In our experience, the machine learning and data processing prototyping phase often happens using Jupyter Notebooks. This can result in disorganized code that performs data processing, feature engineering, model definition, training and testing - all into the same notebook. Experiments can become hard to maintain and reproduce, possibly leading to wrong results and a lot of additional work. A messy ML pipeline makes it much harder also to deploy it in the Cloud for scale up and production.

Kubeflow is becoming the standard de-facto platform when dealing with large scale machine learning workflows on top of Kubernetes. It is designed to simplify the setup, deployment and monitoring of Machine Learning jobs both in the Cloud and on-premises clusters, leveraging on and ecosystem of Cloud Native components. Together with the newly introduced Kubeflow Pipelines (KFP) platform, Kubeflow can help data scientist and software engineers to better organize end-to-end Machine Learning projects, from on-premise prototyping to scaling up in the Cloud and production serving.

---

Despite being designed to be simple and accessible, Kubeflow Pipelines' Python SDK can still result difficult for ML practitioners without a software engineering background. Many data scientists, especially in the research field, come from a physics or maths backgrounds, often lacking the engineering expertise to develop complex deployment and infrastructure setup procedures.

Converting an existing Jupyter Notebook - running on-prem, even on a laptop - to a KFP pipeline, can become a challenging task when dealing with complex workflows. 


#### Kale: a simpler KFP interface designed for Data Scientists

Kale was designed to address these difficulties by providing a tool to simplify the deployment process of a Jupyter Notebook into Kubeflow Pipelines workflows. Translating Jupyter Notebook directly into a KFP pipeline ensures that all the processing building blocks are well organized and independent from each other, while also leveraging on the experiment tracking and workflows organization provided out-of-the-box by Kubeflow.

The main idea behind Kale is to exploit the built-in Jupyter [tagging feature](https://jupyter-notebook.readthedocs.io/en/stable/changelog.html#cell-tags) (in JupyterLab with [this](https://github.com/jupyterlab/jupyterlab-celltags) extension) to:

1. Assign code cells to specific pipeline components
2. Merge together multiple cells into a single pipeline component
3. Define the (execution) dependencies between them

<a href="https://raw.githubusercontent.com/kubeflow-kale/kubeflow-kale.github.io/master/assets/imgs/jupy-to-kfp.png" target="_blank">
  <img src="https://raw.githubusercontent.com/kubeflow-kale/kubeflow-kale.github.io/master/assets/imgs/jupy-to-kfp.png" alt="Kubeflow Kale Deployment - From Jupyter Notebook to KFP pipeline"/>
</a>

Kale takes as input the tagged Jupyter Notebook and generates a standalone Python script that defines the KFP pipeline using [*lightweight components*](https://www.kubeflow.org/docs/pipelines/sdk/lightweight-python-components/), based on the cells annotations. 

---

One question one might raise is: *How does Kale manage to resolve the data dependencies, thus making the notebook variables available throughout the pipeline execution?*

Kale runs a series of static analyses over the Notebook's source Python code to detect where variables and objects are first declared and then used. In this way, Kale creates an internal graph representation describing the data dependencies between the pipeline steps. Using this knowledge, Kale injects code at the beginning and at the end of each component to marshal these objects into a shared PVC during execution. Both marshalling and provisioning of the shared PVC is completely transparent to the user.

<a href="https://raw.githubusercontent.com/kubeflow-kale/kubeflow-kale.github.io/master/assets/imgs/pvc-lifecycle.png" target="_blank">
  <img src="https://raw.githubusercontent.com/kubeflow-kale/kubeflow-kale.github.io/master/assets/imgs/pvc-lifecycle.png" alt="Kubeflow Kale Deployment - PVC Lifecycle"/>
</a>

Kale's internal marshalling module is very flexible in that it can despatch variable's serialization to native functions at runtime, based on the object's data type. This happens also for de-serialization, by reading the file's extension. When the object type is not mapped to a native serialization function, Kale falls back to using the `dill` package, a performant general purpose serialization Python library (superset of `pickle`). guaranteeing maximum performance in terms of disk space and computation time.

## Modularity and Flexibility

Kale's output is an executable self-contained Python script, written with the KFP Python DSL to declare lightweight components that ultimately become the steps of the pipeline. All the required code generation is done via Jinja2 templates - generating Python with Python!

This solution makes Kale very flexible and easy to update to new SDK APIs and breaking changes. All the modules that read the annotated notebook, resolve data dependencies and build the execution graph are independent of KFP and Kubeflow in general. Ironically, Kale could be adapted pretty easily to any workflow runtime that makes available a Python SDK to define workflows in terms of Python functions.

## Deploy a Notebook from Jupyter UI

Kale can be installed in a Python environment and executed from CLI, receiving as input the annotated Jupyter Notebook and the endpoint of the desired KFP deployment. With a single CLI call Kale converts the annotated notebook and deploys the generated executable Python script (describing the KFP pipeline) to the KFP endpoint.

To simplify and streamline the deployment process even further, Kale provides a JupyterLab extension to enable deployment *directly* from the JupyterLab UI. In this way, a data scientist can go from prototyping on a local laptop to a Cloud deployment without ever interacting with the command line.

<a href="https://raw.githubusercontent.com/kubeflow-kale/kubeflow-kale.github.io/master/assets/imgs/jp-ui.png" target="_blank">
  <img src="https://raw.githubusercontent.com/kubeflow-kale/kubeflow-kale.github.io/master/assets/imgs/jp-ui.png" alt="Kubeflow Kale Deployment - JupyterLab Extension"/>
</a>


We are currently working on improving the user experience, adding a Kubeflow specific panel in the JupyterLab left area, making it possible to specify pipeline names, description, data volumes and more directly from UI. Stay tuned for updates in the near future!


## Installation and Usage

For more information about the project, installation and usage documentation, head over to Kale [main repository](https://github.com/kubeflow-kale/kale)

For examples and use cases showcasing Kale in data science pipelines, head over to the [Examples Repository](https://github.com/kubeflow-kale/examples)
