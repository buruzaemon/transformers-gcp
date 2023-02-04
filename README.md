# transformers-gcp

Running with Transformer models on Google Cloud Platform

## Set-up

This covers:
1. setting up a new project on Google Cloud Platform
1. creating a new VM instance for deep learning on Linux on Compute Engine
1. preparing an environment for JupyterLab specifically for working with Transformers

Note that we are _not_ using the User-managed Notebooks feature of Vertex AI. The reason for that is the version/configuration of JupyterLab used in Vertex AI is fixed and not easily edited. When you run any of the `transformers` API code in Vertex AI's notebooks, you will see errors for "Error: Model not found" and you will not be able to load pretrained models, etc., much less use them. Hence, we will be setting up our own JupyterLab environment.

These are my notes from November 2022.

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
    * NOTE that you may have to first upgrade to a paid account to be able to change your quota settings. You can do that by navigating to your Billing Overview, and looking for that button to Upgrade.
    * IF you decide to use another GPU like, say, the NVIDIA A100, then you will also have to request a quota increase from 0 to 1 for the NVIDIA A100 GPUS quota for your target region as well.

At this point, you will have a brand-new project on GCP. 

The next step is create a new Virtual Machine for deep learning under the Compute Engine service we just enabled.

## Create a new Virtual Machine instance

1. In the left-hand navigation of GCP, go to Compute Engine > VM Instance
    * Click the Create Instance button
    * Enter appropriate values for:
        1. Name
        1. Labels(?)
        1. Region ... select a region + zone that
           * that allows for the machine type below
           * is energy-efficient
           * matches your budget expectations
    * Machine configuration: GPU
        1. GPU type: NVIDIA Tesla T4
        1. Number of GPUs: 1
    * Machine type: n1-standard-8 (8 vCPU, 30 GB memory); 30 GB for whisper!
    * Boot disk
        1. In order to have the NVIDAI CUDA drivers installed for us, click CHANGE to switch boot disk to 
           * Operating System: Deep Learning on Linux
           * Version: Debian 10 based Deep Learning VM (with Intel MK) M99; Base CUDA 11.3, Deep Learning VM with CUDA 11.3 preinstalled
           * Boot disk type: Balanced persistent disk
           * Size (GB): 100 (matches the specs you would see for a User-managed Notebook on Vertex AI)
    * Firewall:
        1. Allow HTTP traffic
        1. Allow HTTPS traffic
    * Click the CREATE button!

## Install the NVIDIA drivers

While the Notebooks in Vertex AI already have the NVIDIA drivers installed, we are opting to not use Vertex AI. Hence, we need to manually install them here.

1. If you aren't already there, go to Compute Engine > VM instances.
2. Start up your newly-created virtual machine
3. After your virtual machine instance is up and running, click on the SSH button to open up a browser-based shell for SSH.
4. In this shell window, first try running:
    * `ps aux | grep -i apt`
    * Make sure that nothing (like root user) is running `apt` for updates, etc. 
    * If you try doing the NVIDIA driver install while root user is doing `apt update` or the like, then you will see errors for not being able to acquire a lock on `apt` and you will not be able to install the NVIDIA driver successfully (c.f. [this](https://itsfoss.com/could-not-get-lock-error/))
5. When you are sure that there are no other users/processes using `apt`, in the SSH shell you will see a message that instructs you on how to install the NVIDIA driver. Follow those instructions.
    * Should those instructions in the SSH shell fail, you can always try the instruction to [Install GPU drivers](https://cloud.google.com/compute/docs/gpus/install-drivers-gpu).
    * It may help to actually uninstall any existing run installations with `sudo /usr/bin/nvidia-uninstall`

## Reserve a static IP address and create Firewall rule

Since we are not using the Vertex AI notebooks, we also need to reserve a static IP address and create a Firewall rule to open up TCP traffic on some port for JupyterLab.

### Reserve a static IP address for your virtual machine
1. In the left-hand navigation in GCP cloud console, go to VPC Network > IP addresses
2. Click on Reserve External Static Address
    * Static IP addresses are reserved per virtual machine instance
3. Enter:
    * Name
    * Description
    * Select Standard for Network Service Tier
    * Select Regional for Type, and select the region of your virtual machine for Region.
    * Select your virtual machine instance for Attached to
    * Click Reserve

*NOTE* if you delete a virtual machine instance with a reserved static IP address, be sure to Release that static IP address afterwards. Otherwise, you will continue to be charged for that static IP adddress even though it is not in use! 

### Create a Firewall rule for JupyterLab traffic in your project
1. While still in VPC Network, go to Firewall.
2. Click on Create Firewall Rule
3. Enter:
    * Name
    * Description
    * Network ... keep default
    * Priority ... keep default value of 1000
    * Direction of traffic ... keep default of Ingress
    * Action on match ... keep default of Allow
    * Targets ... select Specified service account
    * Service account scope ... keep default of In this project
    * Target service account ... select your service account
    * Source filter ... keep default of IPv4 ranges
    * Source IPv4 ranges ... enter 0.0.0.0/0
    * Protocols and ports ... choose Specified protocols and ports
    * Select TCP, and enter your favorite port for Jupyter (the default is port 8888)
    * Click the Create button

## Set up GPU performance monitoring

### Download & install GPU Utilization Metric Agent

c.f. [Monitoring GPU Performance on Linux VMs](https://cloud.google.com/compute/docs/gpus/monitor-gpus)

1. Start up an SSH shell to your running virtual machine instance.
2. Create a working directory for the utility code we will be fetching with `git`; change to that working directory.
    * `sudo mkdir -p /opt/google`
    * `cd /opt/google`
3. Download the script for setting up GPU performance monitoring from GitHub.
    * `sudo git clone https://github.com/GoogleCloudPlatform/compute-gpu-monitoring.git`
4. Move to the compute-gpu-monitoring/linux subdirectory.
    * `cd /opt/google/compute-gpu-monitoring/linux`
5. Prepare to create a `venv` virtual environment
    * `sudo apt-get install python3-venv`
6. Create the virtual environment; install the packages.
    * `sudo python3 -m venv venv`
    * `sudo venv/bin/pip install wheel`
    * `sudo venv/bin/pip install -Ur requirements.txt`
7. Start the agent on system boot.
    * `sudo cp /opt/google/compute-gpu-monitoring/linux/systemd/google_gpu_monitoring_agent_venv.service /lib/systemd/system`
    * `sudo systemctl daemon-reload`
    * `sudo systemctl --no-reload --now enable /lib/systemd/system/google_gpu_monitoring_agent_venv.service`
8. Verify that the new GPU monitoring service has been installed correctly, and is presently running.
    * `sudo systemctl list-units | grep gpu`
    * You should see that `google_gpu_monitoring_agent_venv.service` for GPU Utilization Metric Agent is `loaded active running`

### Create a dashboard for GPU monitoring

1. In the left-hand navigation, go to Monitoring > Metrics explorer
2. Click on SELECT A METRIC, and choose VM Instance > Custom > `custom/instance/gpu/utilization`
3. Confirm that you can actually see the GPU utilization chart
4. Now go to Dashboards
5. Click Create Dashboard
6. Enter an appropriate and easily recognizable name for this dashboard which we will use to specifically monitor all available GPU metrics
7. Now return to Metrics explorer
8. Using step 2 above, select all of the available GPU metrics one-by-one, and use the Save Chart button to save this metric to our newly-created dashboard. Set up metrics in our dashboard for:
    * `custom/instance/gpu/utilization`
    * `custom/instance/gpu/memory_utilization`
    * `custom/instance/gpu/memory_total`
    * `custom/instance/gpu/memory_free`
    * `custom/instance/gpu/temperature`
10. Now view our dashboard for GPU monitoring.


### Configure Jupyter Notebook

1. Create a configuration file for Jupyter notebook:
    * `jupyter notebook --generate-config`
2. Add the following configurations right under the line with `c = get_config()` (about line 3 or so)...
    * `c.ServerApp.ip = "*"`
    * `c.ServerApp.port = 8888`
    * `c.ServerApp.open_browser = False`
3. In order to deal with those `Could not determine jupyterlab build status without nodejs` messages when starting up Jupyter Lab, do this:
    * `conda install -c conda-forge nodejs`
    
