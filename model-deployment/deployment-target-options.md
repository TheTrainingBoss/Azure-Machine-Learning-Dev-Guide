# Overview of deployment target options

After you have trained your machine learning model and evaluated it to the point where you are ready to use it outside your own development or test environment, you need to deploy it somewhere. Azure Machine Learning service simplifies this process. You can use the service components and tools to register your model and deploy it to one of the available compute targets so it can be made available as a web service in the Azure cloud, or on an IoT Edge device.

## General deployment process

Regardless of which framework you use to deploy a model, like Azure Machine Learning service, you generally do the following to deploy a model:

- Get the model file (any format)
- Create a scoring script (.py)
- Optionally create a schema file describing the web service input (.json)
- Create a real-time scoring web service
- Call the web service from your applications
- Repeat the process each time you re-train the model

![Image signifies the typical model deployment process.](media/model-deployment-process.png 'Typical model deployment process')

This process is very time-consuming if done manually, especially in creating the real-time scoring web service.

## Deploying with Azure Machine Learning service

Azure Machine Learning service simplifies this process by providing tools that help automate these steps. The series of steps using Azure Machine Learning fall in line with the deployment process above, but most of the work is done for you. You follow these steps as part of the deployment process, regardless of whether you are using your own models, or models you obtain from somewhere else.

![Deployment workflow for Azure Machine Learning.](media/aml-deployment-process.png 'Deployment workflow for Azure Machine Learning')

1. **Register the model** in a registry hosted in your Azure Machine Learning Service workspace
2. **Prepare to deploy** by creating a scoring file, specifying assets, usage, and a compute target
3. **Use** the model in a web service in the cloud, on an IoT device, or for analytics with Power BI
4. **Monitor and collect data**
5. **Update** a deployment to use a new image

The remainder of this article will focus on your deployment target options.

## Available compute targets

The following compute targets, or compute resources, can be used to host your web service deployment.

| Compute target                                                    | Usage                     | Description                                                                                |
| ----------------------------------------------------------------- | ------------------------- | ------------------------------------------------------------------------------------------ |
| [Local web service](#local-web-service)                           | Testing/debug             | Good for limited testing and troubleshooting.                                              |
| [Azure Kubernetes Service (AKS)](#azure-kubernetes-service-aks)   | Real-time inference       | Good for high-scale production deployments. Provides autoscaling, and fast response times. |
| [Azure Container Instances (ACI)](#azure-container-instances-aci) | Testing                   | Good for low scale, CPU-based workloads.                                                   |
| [Azure Machine Learning Compute](#azure-machine-learning-compute) | (Preview) Batch inference | Run batch scoring on serverless compute. Supports normal and low-priority VMs.             |
| [Azure IoT Edge](#azure-iot-edge)                                 | (Preview) IoT module      | Deploy & serve ML models on IoT devices.                                                   |

## Local web service

Deploy locally to quickly test your model image (Docker), or for troubleshooting purposes. To do this, you must have [**Docker installed**](https://docs.docker.com/install/) on your local machine. Docker must be running before you deploy a local web service. This is the recommended compute target if you have problems deploying a model to Azure Kubernetes Service (AKS) or Azure Container Instances (ACI).

Local web service deployments are not supported for production scenarios. If you need to deploy to production web services, the recommended target is [AKS](#azure-kubernetes-service-aks) for high-scale production workloads. For low-scale, CPU-based workloads, use [ACI](#azure-container-instances-aci).

The following sample deploys a model (contained in the `model` variable) as a local web service:

```python
from azureml.core.model import InferenceConfig
from azureml.core.webservice import LocalWebservice

# Create inference configuration. This creates a docker image that contains the model.
inference_config = InferenceConfig(runtime= "python",
                                   execution_script="score.py",
                                   conda_file="myenv.yml")

# Create a local deployment, using port 8890 for the web service endpoint
deployment_config = LocalWebservice.deploy_configuration(port=8890)
# Deploy the service
service = Model.deploy(ws, "mymodel", [model], inference_config, deployment_config)
# Wait for the deployment to complete
service.wait_for_deployment(True)
# Display the port that the web service is available on
print(service.port)
```

You can work with the service just as you would if the compute target were ACI or AKS.

During local testing, you may need to update the `score.py` file to add logging or attempt to resolve any problems that you've discovered. To reload changes to the `score.py` file, use `reload()`. For example, the following code reloads the script for the service, and then sends data to it. The data is scored using the updated `score.py` file:

```python
service.reload()
print(service.run(input_data=test_sample))
```

The script is reloaded from the location specified by the `InferenceConfig` object used by the service.

To change the model, Conda dependencies, or deployment configuration, use `update()`. The following example updates the model used by the service:

```python
service.update([different_model], inference_config, deployment_config)
```

## Azure Kubernetes Service (AKS)

For large-scale production workloads, it is best to deploy your model to AKS.

[Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/) makes it simple to deploy a managed Kubernetes cluster in Azure. AKS reduces the complexity and operational overhead of managing Kubernetes by offloading much of that responsibility to Azure. As a hosted Kubernetes service, Azure handles critical tasks like health monitoring and maintenance for you. The Kubernetes masters are managed by Azure. You only manage and maintain the agent nodes. As a managed Kubernetes service, AKS is free - you only pay for the agent nodes within your clusters, not for the masters.

When you deploy your model to AKS as a target, all you are required to do is provision the AKS service in Azure. Once created, Azure Machine Learning service manages the AKS cluster for you, meaning, you do not need to manage and maintain the agent nodes.

Creating or attaching an AKS cluster is a one time process for your workspace. You can reuse this cluster for multiple deployments. If you delete the cluster or the resource group that contains it, you must create a new cluster the next time you need to deploy.

You can either create the [AKS cluster yourself](https://docs.microsoft.com/azure/aks/kubernetes-walkthrough-portal?view=azure-cli-latest), or use the Azure Machine Learning SDK to do it, specifying parameters such as VM size and agent count. Using the SDK helps you automate creating the cluster as part of your overall deployment pipeline.

This example demonstrates creating an AKS cluster with the Azure Machine Learning SDK:

```python
from azureml.core.compute import AksCompute, ComputeTarget

# Use the default configuration (you can also provide parameters to customize this)
prov_config = AksCompute.provisioning_configuration()

aks_name = 'myaks'
# Create the cluster
aks_target = ComputeTarget.create(workspace = ws,
                                    name = aks_name,
                                    provisioning_configuration = prov_config)

# Wait for the create process to complete
aks_target.wait_for_completion(show_output = True)
```

Once you have created the AKS cluster, you can now deploy to it using the SDK:

```python
aks_target = AksCompute(ws,"myaks")
deployment_config = AksWebservice.deploy_configuration(cpu_cores = 1, memory_gb = 1)
service = Model.deploy(ws, "aksservice", [model], inference_config, deployment_config, aks_target)
service.wait_for_deployment(show_output = True)
print(service.state)
print(service.get_logs())
```

You can also deploy to the AKS cluster using the CLI:

```bash
az ml model deploy -ct myaks -m mymodel:1 -n aksservice -ic inferenceconfig.json -dc deploymentconfig.json
```

Finally, you can use the [Visual Studio Code extension to deploy to AKS](https://docs.microsoft.com/en-us/azure/machine-learning/service/how-to-vscode-tools#deploy-and-manage-models).

## Azure Container Instances (ACI)

For low-scale, CPU-based workloads or testing, deploy to ACI.

[Azure Container Instances](https://docs.microsoft.com/en-us/azure/container-instances/) offers the fastest and simplest way to run a container in Azure, without having to manage any virtual machines and without having to adopt a higher-level service. As opposed to AKS, you use ACI to run isolated containers. ACI is good for smaller or short-term workloads, since you only pay for the time they are up and running. They are also very fast to start, usually within seconds, and delete when you no longer needed. As a point of comparison, AKS clusters are meant to host long-running web services that can scale out to meet heavy workload requirements, and scale back in during lighter workloads.

Use Azure Container Instances for deploying your models as a web service if one or more of the following conditions is true:

- You need to quickly deploy and validate your model.
- You are testing a model that is under development.

To see quota and region availability for ACI, see the [Quotas and region availability for Azure Container Instances](https://docs.microsoft.com/azure/container-instances/container-instances-quotas) article.

Unlike deploying to AKS, you do not need to create ACI containers in advance because they are created on the fly. This means you can go straight to deploying to ACI, as in this deployment script example using the SDK:

```python
deployment_config = AciWebservice.deploy_configuration(cpu_cores = 1, memory_gb = 1)
service = Model.deploy(ws, "aciservice", [model], inference_config, deployment_config)
service.wait_for_deployment(show_output = True)
print(service.state)
```

You can also deploy to ACI using the CLI:

```bash
az ml model deploy -m sklearn_mnist:1 -n aciservice -ic inferenceconfig.json -dc deploymentconfig.json
```

Finally, you can use the [Visual Studio Code extension to deploy to ACI](https://docs.microsoft.com/en-us/azure/machine-learning/service/how-to-vscode-tools#deploy-and-manage-models).

## Azure Machine Learning compute

If you need to make predictions on large quantities of data asynchronously, or if you require intensive compute for scoring, you can use Azure Machine Learning service to make batch predictions. Making batch predictions (or batch scoring) provides a cost-effective inference with unparalleled throughput for asynchronous applications. If you are trying to perform inference on data sets up to terabytes of data, requiring high throughput, performing batch processing using the Azure Machine Learning compute target is your best bet.

When you perform batch scoring, you store the results output to a file store, like [Azure Files](https://docs.microsoft.com/azure/storage/files/storage-files-introduction) or [Blob storage](https://docs.microsoft.com/azure/storage/blobs/storage-blobs-introduction), either of which are created for you when you provision your Azure Machine Learning service workspace. Whereas, with real-time scoring using a web service-deployed model, you are making low-latency requests by sending small amounts of data and receiving a score immediately as an output from the service.

## Azure IoT Edge

If you need to perform analytics on a Linux hardware device that lives "on the edge", that is, not in the cloud but close to where your IoT hardware or sensors live, you can deploy your models to an Azure IoT Edge device. One such use case is you want to respond to emergencies as quickly as possible by running anomaly detection workloads at the edge.

[Azure IoT Edge](https://docs.microsoft.com/en-us/azure/iot-edge/about-iot-edge) is an Internet of Things (IoT) service that builds on top of [Azure IoT Hub](https://docs.microsoft.com/en-us/azure/iot-hub/about-iot-hub). This service is meant for customers who want to analyze data on devices, a.k.a. "at the edge", instead of in the cloud. By moving parts of your workload to the edge, your devices can spend less time sending messages to the cloud and react more quickly to changes in status.

When you deploy to an Azure IoT Edge device using Azure Machine Learning service, you are deploying your model into an **IoT Edge module**, which is a Docker-compatible container. This is not unlike deploying to AKS or an ACI container. Multiple modules can be configured to communicate with each other, creating a pipeline of data processing. You can develop custom modules or package certain Azure services into modules that provide insights offline and at the edge.

## Next steps

Please see the following additional references for deploying to different compute targets:

- [Reference link]()
- [Reference link]()

Read next: [Overview of Real-time inferencing](./real-time-inferencing.md)