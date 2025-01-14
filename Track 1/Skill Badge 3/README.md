## Task 1: Create development VPC manually


Make sure you create all resources in the us-east1 region and `us-east1-b` zone.

1. In the Google Cloud Console, navigate to **VPC network > VPC networks**
2. Click on Create VPC network.
3. Enter `griffin-dev-vpc` to the Name field.
4. Select Custom for the Subnet creation mode.
5. Add `griffin-dev-wp` subnet with the following
    1. Name: `griffin-dev-wp`
    2. Region: `us-east1`
    3. IP address range: `192.168.16.0/20`
6. Click **+ Add subnet** and add `griffin-dev-mgmt` subnet with the following
    1. Name: `griffin-dev-mgmt`
    2. Region: `us-east1`
    3. IP address range: `192.168.32.0/20`
7. Click **Create**.


## Task 2: Create production VPC manually


1. In the Google Cloud Console, navigate to **VPC network > VPC networks**
2. Click on Create VPC network.
3. Enter `griffin-prod-vpc` to the Name field.
4. Select Custom for the Subnet creation mode.
5. Add `griffin-prod-wp` subnet with the following
    1. Name: `griffin-dev-wp`
    2. Region: `us-east1`
    3. IP address range: `192.168.48.0/20`
6. Click **+ Add subnet** and add `griffin-prod-mgmt` subnet with the following
    1. Name: `griffin-dev-mgmt`
    2. Region: `us-east1`
    3. IP address range: `192.168.64.0/20`
7. Click **Create**.


## Task 3: Create bastion host


1. In the Cloud Console, navigate to **Compute Engine > VM instances**.
2. Click **Create**.
3. Use the following parameters to create the bastion host:
    * Name: `griffin-dev-db`
    * Region: `us-east1`
4. Expand the **Management, security, disks, networking, sole tenancy** section.
5. In the **Networking** tab, add `bastion` to the Network tags.
6. Click Add network interface, make sure that you set up two Network interfaces,
    * griffin-dev-mgmt
    * griffin-prod-mgmt
7. Click **Create**.
8. Navigate to **VPC network > Firewall**.
9. Click **CREATE FIREWALL RULE**.
10. Configure the rule with the following parameters:
    * Name:`allow-bastion-dev-ssh`
    * Network:`griffin-dev-vpc`
    * Targets: `bastion`
    * Source IP ranges: `192.168.32.0/20`
    * Protocols and ports: tcp: `22`
11. Click **CREATE**.
12. Click **CREATE FIREWALL RULE** again.
13. Configure another rule with the following parameters:
    * FieldValueName: `allow-bastion-prod-ssh`
    * Network: `griffin-prod-vpc`
    * Targets: `bastion`
    * Source IP ranges: `192.168.64.0/20`
    * Protocols and ports: `tcp: 22`
14. Click **CREATE**.


## Task 4: Create and configure Cloud SQL instance


1. In the Cloud Console, navigate to **SQL**.
2. Click **CREATE INSTANCE**.
3. Click Choose **MySQL**.
4. Use the following parameters to create the instance:
    * Name:`griffin-dev-db`
    * Region:`us-east1`
    * Zone:`us-east1-b`
    * Root password:e.g. 12345678
    * _Note: In real practice, you must set a strong password._
5. Click **Create**.
6. Click the `griffin-dev-db` in the SQL pane after it has created.
7. Under Connect to this instance, click on **Connect using Cloud Shell**.
8. Go back to the Cloud Shell, run:
    ```
    gcloud sql connect griffin-dev-db --user=root --quiet
    ```
9. Enter the Root password generated in Step 4.
10. In the SQL console, run the following query to create the wordpress database:
    ```
    CREATE DATABASE wordpress;
    GRANT ALL PRIVILEGES ON wordpress.* TO "wp_user"@"%" IDENTIFIED BY "stormwind_rules";
    FLUSH PRIVILEGES;
    ```
11. Enter `exit` to quit the SQL shell.


## Task 5: Create Kubernetes Cluster


Create a 2 node cluster (n1-standard-4) called `griffin-dev`, in the `griffin-dev-wp` subnet, and in the zone `us-east1-b`

1. In the Cloud Console, navigate to **Kubernetes Engine > Clusters**.
2. Click **Create cluster**.
3. In the Cluster basics tab, configure:
    * Name: `griffin-dev`
    * Zone: `us-east1-b`
4. In the left pane, click **default-pool** under **NODE POOLS** and setNumber of nodes: `2`
5. Click **Nodes** Under **default-poo**l, and setMachine type: `n1-standard-4`
6. Go to the Network tab, set
    * Network: griffin-dev-vpc
    * Node subnet: griffin-dev-wp
7. Click **Create**.


## Task 6: Prepare the Kubernetes Cluster


1. In the Cloud Shell, use the following command to copy the files for the Kubernetes:
    ```
    gsutil cp -r gs://cloud-training/gsp321/wp-k8s ~/
    ```
2. Open `wp-k8s/wp-env.yaml` with the Cloud Shell Editor.
    ```
    cd ~/wp-k8s
    edit wp-env.yaml
    ```
3. Replace `username_goes_here` and `password_goes_here` to `wp-user` and `stormwind_rules`, respectively.
4. Save the file change.
5. After the Kubernetes cluster has created, click on the **Connect** button.
6. Run the following command to connect the cluster:
    ```
    gcloud container clusters get-credentials griffin-dev --zone us-east1-b –-project [Your Project ID]
    ```
7. Deploy the configuration to the cluster using: `kubectl apply -f wp-env.yaml`
8. Use the command below to create the key, and then add the key to the Kubernetes environment:
    ```
    gcloud iam service-accounts keys create key.json \
        --iam-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
    kubectl create secret generic cloudsql-instance-credentials \
        --from-file key.json
    ```


## Task 7: Create a WordPress Deployment


1. Open `wp-k8s/wp-deployment.yaml` with the Cloud Shell Editor `cd ~/wp-k8s edit wp-deployment.yaml`
2. Replace `YOUR_SQL_INSTANCE` with `griffin-dev-db`’s Instance connection name.
3. Save the file change.
4. Go back to the Cloud Shell, run the following commands:
    ```
    kubectl create -f wp-deployment.yaml 
    kubectl create -f wp-service.yaml
    ```
5. Copy the External endpoints of the deployed **wordpress** service and open it in your browser.


## Task 8: Enable monitoring

1. Go back to the Cloud Console, and navigate to **Monitoring**.
2. In the Monitoring console, click **Uptime checks** in the left pane.
3. Click **CREATE UPTIME CHECK**.
4. Configure using the following parameters:
    * Title: `WordPress Uptime`
    * Check Type: `HTTP`
    * Resource Type: `URL`
    * Hostname: `YOUR-WORDPRESS_ENDPOINT`
    * Path: `/`
5. Click **TEST**.
6. Click **SAVE** if there is no error.


## Task 9: Provide access for an additional engineer

1. In the Cloud Console, navigate to **IAM & Admin > IAM**.
2. Click **+ADD**.
3. In the Add members to … pane, copy and paste the **second user account** for the lab to the **New members** field.
4. In the Role dropdown, select **Project > Editor**.
5. Click **SAVE**.
