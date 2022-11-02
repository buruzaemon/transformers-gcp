# transformers-gcp

Running with Transformer models on Google Cloud Platform

## Set-up

This covers:
1. setting up a new project on Google Cloud Platform
1. creating a new VM instance for deep learning on Linux on Compute Engine
1. preparing an environment for JupyterLab specifically for working with Transformers

Note that we are _not_ using the User-managed Notebooks feature of Vertex AI. The reason for that is the version/configuration of JupyterLab used in Vertex AI is fixed and not easily edited. When you run any of the `transformers` API code in Vertex AI's notebooks, you will see errors for "Error: Model not found" and you will not be able to load pretrained models, etc., much less use them. Hence, we will be setting up our own JupyterLab environment.

## Create a new project

1. Create New Project
    * Go to the Manage resources page in Google Cloud console
    * Enter a Project name.
    * Select a Location.
        * No organization -> select this to create the new project as the top level of its own resource hierarchy.
    * Click that Create button
    * Click the Refresh icon update the Manage resource listing to see your new project
2. Enable the Compute Engine API to create/run virtual machines on GCP
    * Go to APIs & Services using the left-navigation in the Google Cloud console
    * Click on +ENABLE APIS AND SERVICE
    * Enter “compute engine api” in the search box
    * In the search results, click on Compute Engine API
    * Click Enable to turn this API on; we need this for using compute vms
3. Increase the quota on GPUs for the Compute Engine API (the default is 0!)
    * Go to IAM & Admin using the left-navigation in the Google Cloud console
    * Go to Quotas to see the list of quotas for your present project
    * Use the filter to find the GPUS-ALL-REGIONS-per-project quota
    * Check the box to the left of the GPUS-ALL-REGIONS-per-project quota item, and click the Edit Quotas link
    * In the panel that should slide out from the right, you will see the quota change request form for Compute Engine API. 
        1. The current limit: 0
        2. Enter 1 for the New limit value; we will be using only a single GPU for this experimental project
        3. Enter a brief but appropriate Request description.
        4. Click the Next button.
    * In the next screen of the Quota change request form, 
        1. Confirm your Name, Email, and Phone number values.
        2. Click the Submit request button
    * It may take several minutes and even up to two business days before you see that GPUS-ALL-REGIONS-per-project quota value go from 0 to 1. Be patient.

At this point, you will have a brand-new project on GCP. 

The next step is create a new Virtual Machine for deep learning under the Compute Engine service we just enabled.
