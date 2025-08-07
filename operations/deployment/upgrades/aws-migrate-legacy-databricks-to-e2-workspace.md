# AWS: Migrate Legacy Databricks to E2 Workspace

### Sign up for new Databricks account

All new accounts are created on the E2 platform. Sign up using your service account (might have to use a new email if it's already used on the legacy workspace) here [https://databricks.com/try-databricks?itm\_data=Homepage-HeroCTA-Trial](https://databricks.com/try-databricks?itm\_data=Homepage-HeroCTA-Trial)

### Update Terraform Variables

4 variables will need to be added to connect to the new Databricks E2 account

| Key                       | Description                                                                                                                                            | Example Value                       |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------- |
| databricksE2Enabled       | yes/no, enables E2 resources to be created                                                                                                             | yes                                 |
| databricksAccountId       | Main account id, found by going to [https://accounts.cloud.databricks.com/workspaces](https://accounts.cloud.databricks.com/workspaces) and logging in | 638396f1-xxxx-xxxx-9aab-ddf61xxxxxx |
| databricksAccountUser     | Email used to sign up for the E2 account                                                                                                               | databricksmaster@westmonroe.com     |
| databricksAccountP        | P used to sign up for the E2 account (mark as sensitive)                                                                                               |                                     |

### Run Terraform

Queue up plan in Terraform workspace. There should be about 19 resources to change, 5 to add, and 2 to destroy. In the plan, most of the resources should have Databricks in the resource names. If anything seems strange or needs verifying, reach out to your DataOps infrastructure resource. If the plan looks good, apply the plan.

### Turn off environment and reach out to Databricks representative

Once the Terraform plan finishes applying, go to ECS and turn off the Core, API, and Agent containers. While the environment is in a state of Databricks workspace limbo, we do not want any processes running in the DataOps environment.

Reach out to your Databricks representative to migrate the DBFS assets from your legacy workspace to the new E2 workspace. You will need to send 3 values:&#x20;

* Databricks Account Id
  * Same value as "databricksAccountId" variable that was added to Terraform in a previous step
* Workspace URL
  * URL of the new E2 workspace created by Terraform. It should be in an output value from the Terraform run called "databricksWorkspaceUrl"
* Workspace ID
  * Navigate to the workspace URL and grab the value in your browser address bar that comes after the ?o=
  * If your workspace URL in the address bar looked like https://dataops-123.cloud.databricks.com/?o=123456789# then the workspace ID would be 123456789

Wait for Databricks to migrate the DBFS assets before moving on to the next step.

### Use migration tool

Use [https://github.com/databrickslabs/migrate](https://github.com/databrickslabs/migrate) to migrate users, notebooks, clusters and the metastore. You will need a Linux machine with Python and Git installed to be able to run the tool. Clone the tool, set up two profiles "oldWS" and "newWS" and run the following commands:

* Migrate users
  * python3 export\_db.py --profile oldWS --users
  * python3 import\_db.py --profile newWS --users
* Migrate metastore
  * python3 export\_db.py --profile oldWS -- metastore --database "\<your-database-name>"
  * python3 import\_db.py --profile newWs --metastore
    * Need to add existing DataOps instance profile to the migration cluster
* Repair metastore tables
  * python3 import\_db.py --profile newWs --get-repair-log
  * Run linked notebook in E2 workspace [https://github.com/databrickslabs/migrate/blob/master/data/repair\_tables\_for\_migration.py](https://github.com/databrickslabs/migrate/blob/master/data/repair\_tables\_for\_migration.py)
* Migrate notebooks
  * python3 export\_db.py --profile oldWS --workspace
  * python3 export\_db.py --profile oldWS --download
  * python3 export\_db.py -- profile oldWS --workspace-acls
  * python3 import\_db.py --profile newWS --workspace --archive-missing
  * python3 import\_db.py --profile newWS --workspace-acls

These are the most common steps needed to migrate a DataOps Databricks workspace from legacy to E2. If there are any other custom components that need migrated, please refer to the migration tool documentation on GitHub or reach out to your DataOps infrastructure resource.

### Turn on environment

Go to ECS and turn on Core, API, and Agent containers. When the site is up, go to a source that has existing inputs, and reset enrichment/output on the most recent input. It should pass and throw no errors about Databricks or S3. If there are errors, please reach out to your DataOps infrastructure resource.
