# Migrating to AWS HealthOmics Workflows

The following are the **MINIMAL** steps needed to migrate and test a FastQC Nextflow nf-core pipeline. 

The default pipeline generated by `nf-core create` includes a sample workflow that calls `FastQC` and `MultiQC`. For a detailed explanation of what this default pipeline includes, see this [tutorial for creating new pipelines with nf-core](https://nf-co.re/docs/contributing/tutorials/creating_with_nf_core).

The following steps shows how to modify the generated template to run on AWS HealthOmics Workflows while retaining as much in-built Nextflow and NF-core functionality as possible. Successful execution of the workflow (with 4 underlying tasks) was verified in `eu-west-2` using the test data documented below.

## Step 0: Assumptions and prerequisites

* All steps below assume you are working from your `$HOME` directory on a Linux or macOS system and should take ~1hr to complete.
* Source for this workflow is in `$HOME/nf-core-taxprofiler`
* Source for the `omx-ecr-helper` CDK app is in `$HOME/omx-ecr-helper`
* The following required software is available on your system
  * [AWS CDK](https://aws.amazon.com/cdk/)
  * [AWS CLI v2](https://aws.amazon.com/cli/)
  * [jq](https://stedolan.github.io/jq/)


## Step 1: Privatize container images

### Create a public registry namespace config file

A copy of the `public_registry_properties.json` [file](../../../../utils/cdk/omx-ecr-helper/lib/lambda/parse-image-uri/public_registry_properties.json) from the `omx-ecr-helper` CDK app was used.

### Run `inspect_nf.py` on the workflow project

The `inspect_nf.py` script is a utility that inspects a Nextflow workflow definition that uses NF-Core conventions and generates the following:

- A file called `container_image_manifest.json` that lists all the container image URIs used by the workflow
- A supplemental config file for Nextflow called `conf/omics.config`

The commands to do the above look like:

```bash
cd /path/to/workflow
python3 ~/amazon-omics-tutorials/utils/scripts/inspect_nf.py
    -n /path/to/public_registry_properties.json \
    --output-config-file conf/omics.config \
    .
```

### Deploy the `omx-ecr-helper` CDK app

Workflows that run in AWS HealthOmics must have containerized tooling sourced from ECR private image repositories. This workflow uses 14 unique container images. The `omx-ecr-helper` is a CDK application that automates converting container images from public repositories like Quay.io, ECR-Public, and DockerHub to ECR private image repositories.

```
cd ~/amazon-omics-tutorials/utils/cdk/omx-ecr-helper
cdk deploy --all
```

### Process the container image manifest with the omx-ecr-helper app

```bash
cd /path/to/workflow
aws stepfunctions start-execution \
    --state-machine-arn arn:aws:states:<aws-region>:<aws-account-id>:stateMachine:omx-container-puller \
    --input file://container_image_manifest.json
```

## Step 2: Modify the workflow

### Modify the main `nextflow.config` file

Add the following to the bottom of the file:

```groovy
includeConfig 'conf/omics.config'
```

### Enable schema validation for HealthOmics related parameters

In the `nextflow_schema.json` file under `definitions.generic_options.properties` add:

```json
"ecr_registry": {
    "type": "string",
    "description": "Amazon ECR registry to use for container images",
    "fa_icon": "fas fa-cog",
    "hidden": true
}
```

The above removes the warning that `ecr_registry` is an unrecognized parameter when the workflow runs

### Create a `parameter-template.json` file

```json
{
    "input": {"description": "Samplesheet with sample locations.",
                "optional": false},
    "fasta": {"description": "S3 path to FASTA file",
            "optional": false}
}
```

These are the parameters needed to run the workflow. 



## Step 3: Testing

The steps below verify that the pipeline will run on HealthOmics using a minimal test dataset. In this case we will use public data available in the eu-west-2 region. 

The input expectes a nf-core csv file `samplesheet.csv` containing the sequences we want to QC (one line per paired-end). We can use the sequences below as an example.

    sample,fastq_1,fastq_2,expected_cells,seq_center
    Sample_9500764,s3://omics-eu-west-2/sample-inputs/9500764/C0L01ACXX.1.5-percent.R1.fastq.gz,s3://omics-eu-west-2/sample-inputs/9500764/C0L01ACXX.1.5-percent.R2.fastq.gz,"5000","Sample Institute"

Upload the CSV file to an S3 bucket (for simplicity we'll reuse the same bucket to store the workflow definition and the workflow output)

    aws s3 cp samplesheet.csv s3://<mybucket>/nf-core/omicsqc/samplesheet.csv

Now we create an input file `input.json` that will point to this samplesheet

    {
            "input": "s3://<mybucket>/nf-core/omicsqc/samplesheet.csv",
            "fasta": "s3://omics-eu-west-2/reference/hg38/Homo_sapiens_assembly38.fasta"
    }


### Bundle and register the workflow with HealthOmics

The following sequence of shell commands does the following:

1. Bundles the workflow definition (excluding the `.git/` folder into a Zip file).
2. Uploads the zip file to an S3 staging location. This is required because the zip bundle exceeds the 4MiB size limit for direct upload to AWS HealthOmics via the AWS CLI.
3. Registers the workflow definition with AWS HealthOmics and waits for it to become active.

```
cd ~

workflow_name="nf-core-fastqc"
( cd ~/${workflow_name} && zip -9 -r "${OLDPWD}/${workflow_name}.zip" . -x "./.git/*")

definition_uri=s3://**<mybucket>**/omics-workflows/${workflow_name}.zip
aws s3 cp ${workflow_name}.zip ${definition_uri}

workflow_id=$(aws omics create-workflow \
    --engine NEXTFLOW \
    --definition-uri ${definition_uri} \
    --name "${workflow_name}-$(date +%Y%m%dT%H%M%SZ%z)" \
    --parameter-template file://${workflow_name}/parameter-template.json \
    --query 'id' \
    --output text
)

aws omics wait workflow-active --id "${workflow_id}"
aws omics get-workflow --id "${workflow_id}" > "workflow-${workflow_name}.json"
```

### Start a test run of the workflow in HealthOmics

```
workflow_name="nf-core-fastqc"
OMICS_WORKFLOW_ROLE_ARN="arn:aws:iam::**<account-id>**:role/**<rolename>**"
aws omics start-run \
    --role-arn "${OMICS_WORKFLOW_ROLE_ARN}" \
    --workflow-id "$(jq -r '.id' workflow-${workflow_name}.json)" \
    --name "test $(date +%Y%m%d-%H%M%S)" \
    --output-uri s3://<mybucket>/omics-output \
    --parameters file://input.json
```

You can then monitor the progress of the workflow via the [AWS HealthOmics Console](https://console.aws.amazon.com/omics/home#/runs). The workflow should complete in ~1hr.

Note the Omics Workflow role should allow read access to the input file and the containers in the registry, and write access to the output location of the workflow. A sample policy is added below.

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject"
                ],
                "Resource": [
                    "arn:aws:s3:::<mybucket>/*",
                    "arn:aws:s3:::omics-<region>/*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::<mybucket>",
                    "arn:aws:s3:::omics-<region>"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:PutObject"
                ],
                "Resource": [
                    "arn:aws:s3:::<mybucket>/*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "logs:DescribeLogStreams",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"
                ],
                "Resource": [
                    "arn:aws:logs:<region>:<account>:log-group:/aws/omics/WorkflowLog:log-stream:*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "logs:CreateLogGroup"
                ],
                "Resource": [
                    "arn:aws:logs:<region>:<account>:log-group:/aws/omics/WorkflowLog:*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "ecr:BatchGetImage",
                    "ecr:GetDownloadUrlForLayer",
                    "ecr:BatchCheckLayerAvailability"
                ],
                "Resource": [
                    "arn:aws:ecr:<region>:<account>:repository/*"
                ]
            }
        ] 


## Congrats!

The above process should take no more than 1-2hrs to complete and at the end you will have successfully run the nf-core based fastqc workflow using AWS HealthOmics. From this point you can further customize the workflow as needed.

