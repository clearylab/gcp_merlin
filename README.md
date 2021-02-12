# Workflow
1. Upload data to Google bucket
2. Create Filestore fileshare instance
3. Deploy cluster
4. Install software on VM
5. Prepare to run MERlin
6. Run MERlin
7. Analyze results, either on UGER or GCP VM instance
8. Move output from mounted fileshare to Google bucket

# Storage on Google Bucket practices
Currently there are 4 buckets. 
1. `gs://merlintest` which contains some small sets of data for testing MERlin on (I may remove these soon), and a subdirectory `keep` that contains files I wanted to keep between cluster/filestore deployment.
2. `gs://old_merfish_data` which only contains the THP1 dataset.
3. `gs://mb1_1505fov` which contains the 1505 field of view mouse brain dataset with focus lock issues.
4. `gs://george_mb` which contains another mouse brain dataset George collected after fixing the focus lock.

Ideally, each project should have it's own bucket associated with it, for ease of changing storage class and permissions as appropriate. Each bucket will contain the data and the output once MERlin is run. Permissions can be set for individual objects within each bucket, or the bucket as a whole. For projects currently in use, create a bucket that is in the 'standard' default storage class. Once you are done, you can change the bucket to an 'archive' storage class or whatever class is appropriate for your needs. Going down in storage class means that accesses will be more expensive, but storage will be cheaper. 

# Create Filestore Fileshare Instance.
In the top left corner of the GCP console, click the navigation menu, navigate to Filestore, and click Create an Instance. Choose us-east1 as the region and us-east-1b as the zone. Use default settings for everything else, choosing whatever file share name and instance ID. Once you create, note the `server_ip`:`/fileshare_name` that is shown. 

# Deploy Cluster 
Edit `slurm-cluster.yaml` and replace the network_storage `server_ip` with the `server_ip` of your filestore instance. Replace the remote_mount with the `/fileshare_name` .

```
export CLUSTER_DEPLOY_NAME="slurm-cluster"
export CLUSTER_NAME="merlin-cluster"
export CLUSTER_REGION="us-east1"
export CLUSTER_ZONE="us-east1-b"

cd slurm_gcp

gcloud deployment-manager deployments     --project="$(gcloud config get-value core/project)"     create $CLUSTER_DEPLOY_NAME     --config ${CLUSTER_DEPLOY_NAME}.yaml
```

You should now see the deployment listed under Deployment Manager in the console.
You may see a warning about disk sizes not matching, but it is fine to ignore the warning. 


# SSH
To SSH into a VM instance from the console:
1. Go to Compute Engine under navigation > VM instances
2. Select the controller and login VMs and press Start/Resume
3. Once started, press the SSH button to launch an SSH session 

The following commands are to SSH into the login node from the Terminal.
```
gcloud compute ssh ${CLUSTER_NAME}-controller     --command "sudo journalctl -fu google-startup-scripts.service"     --zone $CLUSTER_ZONE

export CLUSTER_LOGIN_NODE=$(gcloud compute instances list \
        --zones ${CLUSTER_ZONE} \
        --filter="name ~ .*login." \
        --format="value(name)" | head -n1)

gcloud compute ssh ${CLUSTER_LOGIN_NODE}     --zone $CLUSTER_ZONE
```

If you attempt to SSH into a VM instance immediately, you will be warned that slurm is currently being installed and a message will be deployed when it is finished. Wait for that message before continuing.

**Remember to suspend VM instances when not using to avoid incurring charges**

# Check mounted disk
I hae been having issues with the mounted disk so make sure that this works, otherwise there's not enough space to write the results of MERlin.
1. Run ```df``` when SSH'ed into the _controller_ VM instance, you should see `/mnt/disks/sec` listed somewhere. If you do not see this, skip to 3. If you do, proceed to 2.
2. If the above works, try writing something to /mnt/disks/sec. For instance, try ```touch a.txt``` . If that works, everything's good. If not, run ```sudo chmod 777 /mnt/disks/sec``` and try to touch a text file again.
3. If 1 does not work, try running `gcloud filestore instances list`. Do you see the fileshare name you created earlier listed? If you do, try step 1 again. I don't know why but sometimes this fixes it. If not, check that you have actually created the filestore instance and put the correct IP and name in the .yaml file.

# Install necessary software
If this is the first time using the instance, you will have to install the necessary software. `installstuff.sh` in the repo contains everything you need to install.
You can upload files inside the SSH window. Run ``` bash installstuff.sh ``` in the home directory

# Prepare to run MERlin
### Configure .merlinenv
Create a .merlinenv file in your home directory as below
```
DATA_HOME=gs://yourbucket
ANALYSIS_HOME=/mnt/disks/sec
PARAMETERS_HOME=/your_parameters
```

Your data should be in a Google bucket. Set the DATA_HOME in the .merlinenv to point to the bucket address of the parent directory of the data folder eg gs://parent if the data is stored like gs://parent/Data/....dax 
If you have installed MERlin from my (zheng-broad) fork as in the installstuff.sh script I have provided, then this should not be a problem. Currently, the official version of MERlin is not configured to accept gs:// addresses.

ANALYSIS_HOME should be in /mnt/disks/sec to use the filestore instance. 

I have provided an example for the parameters directory in test_parameters.

### Configure parameters. 
#### Analysis parameters
I recommend these following parameters in the analysis folder of the parameters, avoiding writing unnecessary images to save space.
a. Decode: "write_decoded_images": false,
b. FiducialCorrelationWarp: "write_aligned_images": false,
c. GenerateMosaic: "separate_files":true 

d. To prevent z-duplicates, I recommend setting the parameters in decode to
* "remove_z_duplicated_barcodes": true,
* "z_duplicate_xy_pixel_threshold": 3

Adjust the analysis .json appropriately and provide your own codebook and dataorganization file. Also, make sure to change the /snakemake files to have appropriate paths.

# Run MERlin 
## If you have ~100 FOVs
runmerlin.sh
```
#!/bin/bash
#SBATCH -n 1
#SBATCH -N 1
#SBATCH -t 7-00:00:00
#SBATCH --mem 2000
#SBATCH --open-mode=append
#SBATCH --output="slurm_outputs/slurm-%A.out"

conda activate merlin_env

export GOOGLE_APPLICATION_CREDENTIALS="/home/zheng/imaging-transcriptomics-biccn-819256ebece6.json"

cd ~

merlin -a run_all.json -m VizgenAlpha.json -o dataorganization.csv -c codebook_M22E1.csv -k snake.
json data_for_Peter
```
Change the paths in the above bash file as appropriate. I put the output for this job in slurm_outputs to avoid cluttering my home folder, you can change that. 
If you are running multiple jobs at once with different data and analysis paths, you can specify those with -e and -s respecively in the MERlin command. You can NOT specify different parameter paths from the .merlinenv, so maintain one parameters directory and change the names of the files as appropriate if you are running multiple MERlin jobs with different parameters.
In particular, merlin/GCP does not allow me to read directly from the bucket without using gsutil to download without exporting the `GOOGLE_APPLICATION_CREDENTIALS` as above. That json is currently in gs://merlintest/keep. Those are my credentials so I'm not sure they'll work for you, See here for how I got those: https://cloud.google.com/docs/authentication/getting-started

Submit the job using `sbatch runmerlin.sh`
Use `squeue` to check current job statuses and `scancel job_id` to cancel a job. See slurm documentation for more commands.

## If you have many FOVs (>1000)
Running MERlin on 1500 FOVs will increase the memory requirements for certain tasks including GenerateMosaic and potentially ExportBarcodes beyond the 7.5GB available in the current configuration. Instead of running MERlin once with one parameters file as above, you should instead run all of the MERlin tasks EXCEPT for those that require more memory following the above steps. There is an example analysis json called `no_generate_mosaic.json` that contains all of the same parameters as `run_all.json` except the GenerateMosaic task. After the Fiducial Correlation Warp task is done, you will launch a separate VM instance with sufficient memory to run the GenerateMosaic task. Make sure to put the parameters directory on the mounted Filestore file share so both VMs can access it.

### New VM Instance
Navigate to the VM instances page and click Create Instance. Set region and zone to us-east1-b. The compute machines used for the other tasks are N1-standard-2, for this instance I found N1-highmem-2 was sufficient for running GenerateMosaic on 1500 FOVs. You can set the Boot disk Operating system to Centos just like the other machines. Make sure the Compute Engine default service account has full access to all Cloud APIs, then Create.

Before running `installstuff.sh`, run
```
sudo yum update &&
sudo yum install -y nfs-utils wget mesa-libGL gcc git
```

Execute `sudo mkdir -p mount-point-directory` replacing mount-point-directory with the path where you want to mount the fileshare instance (preferrably the same path that it is mounted on in the other VMs for the main task). Then run `sudo mount ip-address:/file-share mount-point-directory` where the ip-address:/file-share is the server IP address and fileshare name for the Filestore instance. Follow the same instructions as before in *Check Mounted Disk* to check that this is properly mounted.

Then run `runmerlin.sh` replacing the parameters.json file with the file that just contains the GenerateMosaic task (example shown in generate_mosaic.json). You may also need to check the file paths for output, as I am directing outputs from the main file submitted via sbatch to ~/slurm_outputs and outputs from the submitted tasks to ~/snake_outputs. Create those directories in the new home directory of this new VM if necessary.
### 

# Analysis
Once you are done, move the output from /mnt/disks/sec to the Google Bucket where the data is using `gsutil -m cp -r current_dir gs://bucket_dir`. You can analyze results either on the VM instance or on UGER by downloading the data from the Google Bucket.

To generate "mosaics" from the barcodes.csv, change the variables at the top of make_merfish_mosaics.py and run.

# Done with everything
Shut down the filestore instance and deployment, deleting all VMs created by it. If you no longer plan on accessing the Google bucket, change its storage level to Archive. 


    
