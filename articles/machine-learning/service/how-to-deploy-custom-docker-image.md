---
title: Deploy models with a custom Docker base image
titleSuffix: Azure Machine Learning service
description: 'Learn how to use a custom Docker base image when deploying your Azure Machine Learning service models. When deploying a trained model, a base container image is deployed to run your model for inference. While Azure Machine Learning service provides a default base image for you, you can also use your own base image.'
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.topic: conceptual
ms.author: jordane
author: jpe316
ms.reviewer: larryfr
ms.date: 08/22/2019
---

# Deploy a model using a custom Docker base image

Learn how to use a custom Docker base image when deploying trained models with the Azure Machine Learning service.

When you deploy a trained model to a web service or IoT Edge device, a package is created which contains a web server to handle incoming requests.

Azure Machine Learning service provides a default Docker base image so you don't have to worry about creating one. You can also use Azure Machine Learning service __environments__ to select a specific base image, or use a custom one that you provide.

A base image is used as the starting point when an image is created for a deployment. It provides the underlying operating system and components. The deployment process then adds additional components, such as your model, conda environment, and other assets, to the image before deploying it.

Typically, you create a custom base image when you want to use Docker to manage your dependencies, maintain tighter control over  component versions or save time during deployment. For example, you might want to standardize on a specific version of Python, Conda, or other component. You might also want to install software required by your model, where the installation process takes a long time. Installing the software when creating the base image means that you don't have to install it for each deployment.

> [!IMPORTANT]
> When you deploy a model, you cannot override core components such as the web server or IoT Edge components. These components provide a known working environment that is tested and supported by Microsoft.

> [!WARNING]
> Microsoft may not be able to help troubleshoot problems caused by a custom image. If you encounter problems, you may be asked to use the default image or one of the images Microsoft provides to see if the problem is specific to your image.

This document is broken into two sections:

* Create a custom base image: Provides information to admins and DevOps on creating a custom image and configuring authentication to an Azure Container Registry using the Azure CLI and Machine Learning CLI.
* Deploy a model using a custom base image: Provides information to Data Scientists and DevOps / ML Engineers on using custom images when deploying a trained model from the Python SDK or ML CLI.

## Prerequisites

* An Azure Machine Learning service workgroup. For more information, see the [Create a workspace](how-to-manage-workspace.md) article.
* The [Azure Machine Learning SDK](https://docs.microsoft.com/python/api/overview/azure/ml/install?view=azure-ml-py). 
* The [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest).
* The [CLI extension for Azure Machine Learning](reference-azure-machine-learning-cli.md).
* An [Azure Container Registry](/azure/container-registry) or other Docker registry that is accessible on the internet.
* The steps in this document assume that you are familiar with creating and using an __inference configuration__ object as part of model deployment. For more information, see the "prepare to deploy" section of [Where to deploy and how](how-to-deploy-and-where.md#prepare-to-deploy).

## Create a custom base image

The information in this section assumes that you are using an Azure Container Registry to store Docker images. Use the following checklist when planning to create custom images for Azure Machine Learning service:

* Will you use the Azure Container Registry created for the Azure Machine Learning service workspace, or a standalone Azure Container Registry?

    When using images stored in the __container registry for the workspace__, you do not need to authenticate to the registry. Authentication is handled by the workspace.

    > [!WARNING]
    > The Azure Container Rzegistry for your workspace is __created the first time you train or deploy a model__ using the workspace. If you've created a new workspace, but not trained or created a model, no Azure Container Registry will exist for the workspace.

    For information on retrieving the name of the Azure Container Registry for your workspace, see the [Get container registry name](#getname) section of this article.

    When using images stored in a __standalone container registry__, you will need to configure a service principal that has at least read access. You then provide the service principal ID (username) and password to anyone that uses images from the registry. The exception is if you make the container registry publicly accessible.

    For information on creating a private Azure Container Registry, see [Create a private container registry](/azure/container-registry/container-registry-get-started-azure-cli).

    For information on using service principals with Azure Container Registry, see [Azure Container Registry authentication with service principals](/azure/container-registry/container-registry-auth-service-principal).

* Azure Container Registry and image information: Provide the image name to anyone that needs to use it. For example, an image named `myimage`, stored in a registry named `myregistry`, is referenced as `myregistry.azurecr.io/myimage` when using the image for model deployment

* Image requirements: Azure Machine Learning service only supports Docker images that provide the following software:

    * Ubuntu 16.04 or greater.
    * Conda 4.5.# or greater.
    * Python 3.5.# or 3.6.#.

<a id="getname"></a>

### Get container registry information

In this section, learn how to get the name of the Azure Container Registry for your Azure Machine Learning service workspace.

> [!WARNING]
> The Azure Container Registry for your workspace is __created the first time you train or deploy a model__ using the workspace. If you've created a new workspace, but not trained or created a model, no Azure Container Registry will exist for the workspace.

If you've already trained or deployed models using the Azure Machine Learning service, a container registry was created for your workspace. To find the name of this container registry, use the following steps:

1. Open a new shell or command-prompt and use the following command to authenticate to your Azure subscription:

    ```azurecli-interactive
    az login
    ```

    Follow the prompts to authenticate to the subscription.

2. Use the following command to list the container registry for the workspace. Replace `<myworkspace>` with your Azure Machine Learning service workspace name. Replace `<resourcegroup>` with the Azure resource group that contains your workspace:

    ```azurecli-interactive
    az ml workspace show -w <myworkspace> -g <resourcegroup> --query containerRegistry
    ```

    [!INCLUDE [install extension](../../../includes/machine-learning-service-install-extension.md)]

    The information returned is similar to the following text:

    ```text
    /subscriptions/<subscription_id>/resourceGroups/<resource_group>/providers/Microsoft.ContainerRegistry/registries/<registry_name>
    ```

    The `<registry_name>` value is the name of the Azure Container Registry for your workspace.

### Build a custom base image

The steps in this section walk-through creating a custom Docker image in your Azure Container Registry.

1. Create a new text file named `Dockerfile`, and use the following text as the contents:

    ```text
    FROM ubuntu:16.04

    ARG CONDA_VERSION=4.5.12
    ARG PYTHON_VERSION=3.6

    ENV LANG=C.UTF-8 LC_ALL=C.UTF-8
    ENV PATH /opt/miniconda/bin:$PATH

    RUN apt-get update --fix-missing && \
        apt-get install -y wget bzip2 && \
        apt-get clean && \
        rm -rf /var/lib/apt/lists/*

    RUN wget --quiet https://repo.anaconda.com/miniconda/Miniconda3-${CONDA_VERSION}-Linux-x86_64.sh -O ~/miniconda.sh && \
        /bin/bash ~/miniconda.sh -b -p /opt/miniconda && \
        rm ~/miniconda.sh && \
        /opt/miniconda/bin/conda clean -tipsy

    RUN conda install -y conda=${CONDA_VERSION} python=${PYTHON_VERSION} && \
        conda clean -aqy && \
        rm -rf /opt/miniconda/pkgs && \
        find / -type d -name __pycache__ -prune -exec rm -rf {} \;
    ```

2. From a shell or command-prompt, use the following to authenticate to the Azure Container Registry. Replace the `<registry_name>` with the name of the container registry you want to store the image in:

    ```azurecli-interactive
    az acr login --name <registry_name>
    ```

3. To upload the Dockerfile, and build it, use the following command. Replace `<registry_name>` with the name of the container registry you want to store the image in:

    ```azurecli-interactive
    az acr build --image myimage:v1 --registry <registry_name> --file Dockerfile .
    ```

    During the build process, information is streamed to back to the command line. If the build is successful, you receive a message similar to the following text:

    ```text
    Run ID: cda was successful after 2m56s
    ```

For more information on building images with an Azure Container Registry, see [Build and run a container image using Azure Container Registry Tasks](https://docs.microsoft.com/azure/container-registry/container-registry-quickstart-task-cli)

For more information on uploading existing images to an Azure Container Registry, see [Push your first image to a private Docker container registry](/azure/container-registry/container-registry-get-started-docker-cli).

## Use a custom base image

To use a custom image, you need the following information:

* The __image name__. For example, `mcr.microsoft.com/azureml/o16n-sample-user-base/ubuntu-miniconda` is the path to a basic Docker Image provided by Microsoft.
* If the image is in a __private repository__, you need the following information:

    * The registry __address__. For example, `myregistry.azureecr.io`.
    * A service principal __username__ and __password__ that has read access to the registry.

    If you do not have this information, speak to the administrator for the Azure Container Registry that contains your image.

### Publicly available base images

Microsoft provides several docker images on a publicly accessible repository, which can be used with the steps in this section:

| Image | Description |
| ----- | ----- |
| `mcr.microsoft.com/azureml/o16n-sample-user-base/ubuntu-miniconda` | Basic image for Azure Machine Learning service |
| `mcr.microsoft.com/azureml/onnxruntime:latest` | Contains the ONNX runtime. |
| `mcr.microsoft.com/azureml/onnxruntime:latest-cuda` | Contains the ONNX runtime and CUDA components. |
| `mcr.microsoft.com/azureml/onnxruntime:latest-tensorrt` | Contains ONNX runtime and TensorRT. |
| `mcr.microsoft.com/azureml/onnxruntime:latest-openvino-myriad` | Contains ONNX runtime and OpenVINO for Myriad-X USB Stick. |
| `mcr.microsoft.com/azureml/onnxruntime:latest-openvino-vadm` | Contains ONNX runtime and OpenVINO for 8 node Myriad-X PCIe card. |

Additional details about the ONNX Runtime base images can be found [here](https://github.com/microsoft/onnxruntime/tree/master/dockerfiles)

> [!TIP]
> Since these images are publicly available, you do not need to provide an address, username or password when using them.

> [!IMPORTANT]
> Microsoft images that use CUDA or TensorRT must be used on Microsoft Azure Services only.

For more information, see [Azure Machine Learning service containers](https://github.com/Azure/AzureML-Containers).

> [!TIP]
>__If your model is trained on Azure Machine Learning Compute__, using __version 1.0.22 or greater__ of the Azure Machine Learning SDK, an image is created during training. To discover the name of this image, use `run.properties["AzureML.DerivedImageName"]`. The following example demonstrates how to use this image:
>
> ```python
> # Use an image built during training with SDK 1.0.22 or greater
> image_config.base_image = run.properties["AzureML.DerivedImageName"]
> ```

### Use an image with the Azure Machine Learning SDK

To use an image stored in the **Azure Container Registry for your workspace**, or a **container registry that is publicly accessible**, set the following [Environment](https://docs.microsoft.com/python/api/azureml-core/azureml.core.environment.environment?view=azure-ml-py) attributes:

+ `docker.enabled=True`
+ `docker.base_image`: Set to the registry and path to the image.

```python
from azureml.core import Environment
# Create the environment
myenv = Environment(name="myenv")
# Enable Docker and reference an image
myenv.docker.enabled = True
myenv.docker.base_image = "mcr.microsoft.com/azureml/o16n-sample-user-base/ubuntu-miniconda"
```

To use an image from a __private container registry__ that is not in your workspace, you must use `docker.base_image_registry` to specify the address of the repository and a user name and password:

```python
# Set the container registry information
myenv.docker.base_image_repository.address = "myregistry.azurecr.io"
myenv.docker.base_image_repository.username = "username"
myenv.docker.base_image_repository.password = "password"
```

After defining the environment, use it with an [InferenceConfig](https://docs.microsoft.com/python/api/azureml-core/azureml.core.model.inferenceconfig?view=azure-ml-py) object to define the inference environment in which the model and web service will run.

```python
from azureml.core.model import InferenceConfig
# Use environment in InferenceConfig
inference_config = InferenceConfig(entry_script="score.py",
                                   environment=myenv)
```

At this point, you can continue with deployment. For example, the following code snippet would deploy a web service locally using the inference configuration and custom image:

```python
from azureml.core.webservice import LocalWebservice, Webservice

deployment_config = LocalWebservice.deploy_configuration(port=8890)
service = Model.deploy(ws, "myservice", [model], inference_config, deployment_config)
service.wait_for_deployment(show_output = True)
print(service.state)
```

For more information on deployment, see [Deploy models with Azure Machine Learning service](how-to-deploy-and-where.md).

### Use an image with the Machine Learning CLI

> [!IMPORTANT]
> Currently the Machine Learning CLI can use images from the Azure Container Registry for your workspace or publicly accessible repositories. It cannot use images from standalone private registries.

When deploying a model using the Machine Learning CLI, you provide an inference configuration file that references the custom image. The following JSON document demonstrates how to reference an image in a public container registry:

```json
{
   "entryScript": "score.py",
   "runtime": "python",
   "condaFile": "infenv.yml",
   "extraDockerfileSteps": null,
   "sourceDirectory": null,
   "enableGpu": false,
   "baseImage": "mcr.microsoft.com/azureml/o16n-sample-user-base/ubuntu-miniconda",
   "baseImageRegistry": "mcr.microsoft.com"
}
```

This file is used with the `az ml model deploy` command. The `--ic` parameter is used to specify the inference configuration file.

```azurecli
az ml model deploy -n myservice -m mymodel:1 --ic inferenceconfig.json --dc deploymentconfig.json --ct akscomputetarget
```

For more information on deploying a model using the ML CLI, see the "model registration, profiling, and deployment" section of the [CLI extension for Azure Machine Learning service](reference-azure-machine-learning-cli.md#model-registration-profiling-deployment) article.

## Next steps

* Learn more about [Where to deploy and how](how-to-deploy-and-where.md).
* Learn how to [Train and deploy machine learning models using Azure Pipelines](/azure/devops/pipelines/targets/azure-machine-learning?view=azure-devops).
