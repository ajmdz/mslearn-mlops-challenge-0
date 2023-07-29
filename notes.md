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

