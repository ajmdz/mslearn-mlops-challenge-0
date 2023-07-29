run `az ml job create --file job.yml --resource-group ... --workspace-name ..`

What went wrong: `compute: azure:amltest` instead of `compute: amltest`

Notes:
* git clone inside compute instance
* register data asset of type `uri_folder` pointing to `/experimentation/data` in workspace blob store


> GitHub is authenticated to use your Azure Machine Learning workspace with a service principal. The service principal is only allowed to submit jobs that use a compute cluster, not a compute instance.