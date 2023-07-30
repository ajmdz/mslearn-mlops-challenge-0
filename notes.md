see: 
* [Create and manage data assets](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-create-data-assets?view=azureml-api-2&tabs=cli)
* [Data concepts in Azure Machine Learning](https://learn.microsoft.com/en-us/azure/machine-learning/concept-data?view=azureml-api-2)
* [Tutorial: Upload, access and explore your data in Azure Machine Learning](https://learn.microsoft.com/en-us/azure/machine-learning/tutorial-explore-data?view=azureml-api-2)

> Use `azureml:` when referring to an existing registered data asset.

> A **service principal** is a security identity that is used to authenticate applications, services, and automation tools when interacting with Azure resources. It provides a way for non-human entities (like applications, scripts, or services) to access resources in an Azure Active Directory (Azure AD) tenant

To give GitHub access to any Azure Machine Learning workspace, you need to create a service principal in Azure. Next, you need to give the service principal access to the Azure Machine Learning workspace in Azure

Create a service principal:
```bash
az ad sp create-for-rbac --name "myML" --role contributor \
                            --scopes /subscriptions/<subscription-id>/resourceGroups/<group-name> \
                            --sdk-auth
```
Use `AZURE_CREDENTIALS` as the name for GitHub secret

# Azure ML jobs
**SPECIFY RESOURCE GROUP AND WORKSPACE WHEN SUBMITTING AML JOBS**

run `az ml job create --file job.yml --resource-group ... --workspace-name ..`

What went wrong: `compute: azure:amltest` instead of `compute: amltest`

Notes:
* git clone inside compute instance
* register data asset of type `uri_folder` pointing to `/experimentation/data` in workspace blob store


> GitHub is authenticated to use your Azure Machine Learning workspace with a service principal. The service principal is only allowed to submit jobs that use a compute cluster, not a compute instance.

 To trigger a GitHub Actions workflow, you can use `on: [pull_request]`. When you use this trigger, your workflow will run whenever the pull request is created.

If you want a workflow to run whenever a pull request is merged, you'll need to use another trigger. **Merging a pull request is essentially a push to the main branch**. So, to trigger a workflow to run when a pull request is merged, use the following trigger in the GitHub Actions workflow:

```yml
on:
  push:
    branches:
      - main
```

> Triggering a workflow by pushing directly to the repo is not considered a best practice. Preferably, you’ll want to review any changes before you build them with GitHub Actions.

> By default, branch protection rules do not apply to administrators. If you’re the administrator of the repo you’re working with, you’ll still be allowed to push directly to the repo.

> To configure the code checks to be required before merging a pull request, your job needs to have a name in the GitHub Actions workflow. You can then find the checks by searching for the job names.


Steps to automate checks for new pull requests:
1. Add linters and unit test jobs to the workflow that is triggered by new pull requests
2. Settings > Branches > add branch protection rules for `main`:
    * Require a pull request before merging
        * Require approvals
    * Require status checks to pass before merging
        * Require branches to be up to date before merging
        * *add status checks from GitHub Actions*
3. Save changes

<hr>

# Work with  environments in GitHub Actions
To work with environments, you'll want to:
* Create environments in your GitHub repository.
* Store credentials to each Azure Machine Learning workspace as an environment secret in GitHub.
* Add required reviewers to environments for gated approval.
* Use environments in your GitHub Actions workflows.

[Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)

> Having separate environments will make it easier to control access to resources. Each environment can then be associated with a separate Azure Machine Learning workspace.

The **development** environment is used for the inner loop:
* Data scientists train the model.
* The model is packaged and registered.

The **staging** environment is used for part of the outer loop:
* Test the code and model with linting and unit testing.
* Deploy the model to test the endpoint.

The **production** environment is used for another part of the outer loop:
* Deploy the model to the production endpoint. The production endpoint is integrated with the web application.
* Monitor the model and endpoint performance to trigger retraining when necessary.

## Use environments in GitHub Actions and add approvals
To give you the opportunity to review the output of the model training in the Azure Machine Learning workspace, you can add an approval for an environment. Whenever a GitHub Actions workflow wants to run a task in a specific environment, the required reviewers will be notified and need to approve the tasks before they'll be run.

To add an approval check within GitHub, navigate to the environment you created:
1. Enable required reviewers.
2. Select the GitHub users you want to enlist as approvers.
3. Save the protection rules.

Whenever a workflow in GitHub Actions wants to deploy to an environment with an approval check, the approvers will get notified that their review is requested.

<hr>

# Deploy a model with GitHub Actions
> Your work as a machine learning engineer, is to establish a process that can automatically deploy the trained model to production.

When you register the model as an **MLflow** model, you can opt for no-code deployment in the Azure Machine Learning workspace. **When you use no-code deployment, you don't need to create the scoring script and environment for the deployment to work**.

When you want to deploy a model, you have a choice between an **online endpoint** for real-time predictions or a **batch endpoint** for batch predictions. *Example: for integrating with a web app expecting a direct response, deploying a model to an **online endpoint** may be more suitable*

 > You can deploy the model manually in the Azure Machine Learning workspace. However, if you expect to deploy more models in the future, you want to easily redeploy the model whenever the model has been retrained. You therefore want to automate the model deployment wherever possible.

 ## [Model Deployment](https://learn.microsoft.com/en-us/training/modules/deploy-model-github-actions/4-model-deployment)
1. Package and register the model
> To register the model, you can point to either a job's output, or to a location in an Azure Machine Learning datastore.
2. [Create an endpoint and deploy the model](https://learn.microsoft.com/en-us/training/modules/deploy-azure-machine-learning-model-managed-endpoint-cli-v2/3-deploy-model-managed-endpoint)
> An endpoint is an HTTPS endpoint that the web app can send data to and get a prediction from. You want the endpoint to remain the same, even after you deploy an updated model to the same endpoint. When the endpoint remains the same, the web app won't need to be updated every time the model is retrained.

### Creating a managed online endpoint:
Include the following parameters in the YAML configuration to create a managed online endpoint:

* `name`: Name of the endpoint. Must be unique in the Azure region.
* `traffic`: (Optional) Percentage of traffic from the endpoint to divert to each deployment. Sum of traffic values must be 100.
* `auth_mode`: Use `key` for key-based authentication. Use `aml_token` for Azure Machine Learning token-based authentication.

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineEndpoint.schema.json
name: mlflow-endpoint
auth_mode: <key/aml_token>
```

To actually create the endpoint:
```bash
az ml online-endpoint create --name diabetes-mlflow -f ./mslearn-aml-cli/Allfiles/Labs/05/mlflow-endpoint/create-endpoint.yml
```

### Deploy a model to an endpoint
To deploy a model, you must have:

* Model files stored on local path or registered model
* Scoring script
* Environment
* Instance type and scaling capacity
  * `instance_type`: VM SKU that will host your deployment instance.
  * `instance_count`: Number of instances in the deployment.

> When you deploy a MLflow model to a managed online endpoint, you don´t need to have the scoring script and environment.

To define the deployment for a MLflow model:
```yaml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineDeployment.schema.json
name: mlflow-deployment
endpoint_name: churn-endpoint
model:
  name: mlflow-sklearn-model
  version: 1
  local_path: model
  model_format: mlflow
instance_type: Standard_F2s_v2
instance_count: 1
```
In this example, we're taking the model files from a local path. The files are all stored in a local folder called `model`. When this YAML is submitted to deploy the model to the endpoint `churn-endpoint`, the model will be registered in the Azure Machine Learning workspace as `mlflow-sklearn-model`, version 1.

To deploy (and automatically register) the model, run the following command:
```bash
az ml online-deployment create --name mlflow-deployment --endpoint diabetes-mlflow -f ./mslearn-aml-cli/Allfiles/Labs/05/mlflow-endpoint/mlflow-deployment.yaml --all-traffic
```

Here, all traffic to the endpoint will be routed to this one deployment because we use `--all-traffic`. If you want to deploy multiple models to the endpoint, you can reconfigure traffic to route incoming requests to the different deployments.

For example, if after testing you want to reroute all traffic to the green deployment, which uses the newest version of the model, use the following command:
```bash
az ml online-endpoint update --name churn-endpoint --traffic "blue=0 green=100"
```

To delete the endpoint and all associated deployments, run the command:
```bash
az ml online-endpoint delete --name churn-endpoint --yes --no-wait
```

See: 
* [Deploy MLflow models to online endpoints](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-deploy-mlflow-models-online-endpoints?view=azureml-api-2&tabs=cli)
* [Guidelines for deploying MLflow models](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-deploy-mlflow-models?view=azureml-api-2&tabs=azureml)
* [CLI (v2) managed online deployment YAML schema](https://learn.microsoft.com/en-us/azure/machine-learning/reference-yaml-deployment-managed-online?view=azureml-api-2)



3. Test the model
> Test the deployed model before integrating the endpoint with the web app. Or before converting all traffic of an endpoint to the updated model. 
You can add a test task to the same workflow as the model deployment task. However, model deployment may take a while to complete. You therefore need to **ensure that the testing only happens when the model deployment is completed successfully**.
