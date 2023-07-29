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