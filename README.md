# cdk-python-module-gitlab-pipeline
A simple GitLab pipeline to package custom CDK Stacks & Constructs as Python modules

## Overview
The code sample shows how to create a python module from custom CDK stacks or constructs. 
A template for GitLab CI/CD is provided in a `gitlab-ci.yml` file to create a build pipeline. Python modules are published from GitLab to AWS CodeArtifact using [Twine](https://docs.gitlab.com/ee/user/packages/pypi_repository/#publish-a-pypi-package-by-using-twine). Developers can consume the modules in their projects by setting the PIP repository index to the CodeArtifact URL. 
Customers want a way to reuse stacks or constructs across projects. Packaging stacks as modules and storing them in CodeArtifact improves reusability,  adoption best practices and speed up development.

## Pre-requisites
Before using the code sample make sure to implement the following pre-requisites. 

* Python: **Follow the [instructions](https://wiki.python.org/moin/BeginnersGuide/Download) for your operating system to install Python.**


* Pip: **Follow the [instructions](https://pip.pypa.io/en/stable/cli/pip_install/) for your operating system to install Pip.**

## Clone this repository

```bash
git clone https://github.com/aws-samples/cdk-python-module-gitlab-pipeline
```
### AWS Setup

1. Install the AWS CLI

Follow the instructions in the [official AWS documentation](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) to install the CLI.

2. Define the required environment variables

The solution works with any AWS account. The account id is retrieved using `aws sts` and the value is assigned to an environment variable. Users should have the permissions to run `aws sts-get-caller-identity` commands from the CLI before executing the steps. 


```bash
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export PYTHON_MODULE_NAME=cdk-python-module 
export DOMAIN_PREFIX=mydomain
export AWS_REGION=us-east-1
export CODE_ARTIFACT_DOMAIN=${DOMAIN_PREFIX}${AWS_ACCOUNT_ID}
export CODE_ARTIFACT_REPO_NAME=${DOMAIN_PREFIX}${PYTHON_MODULE_NAME}
export GITLAB_HOST=mygitlabhost.xyz.com
export GITLAB_REPO_NAME=myuser/cdk-python-module
```

**Notes:** 

* Replace the value of GITLAB_HOST with your self-hosted GitLab instance. You do not need to set that value if you are using gitlab.com
* The GITLAB_REPO_NAME variable is in the format *<OWNER>/<NAMESPACE>/<REPO> ; replace the values before you run the steps. 
* In order to use the solution you will define the following environment variables. You can customize the DOMAIN_PREFIX, CODE_ARTIFACT_REPO_NAME, PYTHON_MODULE_NAME and AWS_REGION to meet the needs of your project.

3. Create an AWS CodeArtifact repository on AWS

[AWS CodeArtifact](https://docs.aws.amazon.com/codeartifact/index.html) is a secure, scalable, and cost-effective artifact management service for software development.

In the next steps we will use the [AWS CLI](https://docs.aws.amazon.com/codeartifact/latest/ug/getting-started-cli.html) to create a python module repository in CodeArtifact. 

**Note:** *You should have the permissions to call `aws codeartifact` before performing the steps below.


```bash
aws codeartifact create-domain --domain ${CODE_ARTIFACT_DOMAIN} --region ${AWS_REGION}
aws codeartifact create-repository --domain ${CODE_ARTIFACT_DOMAIN} --repository ${CODE_ARTIFACT_REPO_NAME} --description "sample repository for python cdk modules" --region ${AWS_REGION}
```

4. Connect to AWS CodeArtifact

Use the command below to [authenticate](https://docs.aws.amazon.com/codeartifact/latest/ug/connect-repo.html) with CodeArtifact. 

```bash
aws codeartifact login --tool pip --repository ${CODE_ARTIFACT_REPO_NAME} --domain ${CODE_ARTIFACT_DOMAIN} --domain-owner ${AWS_ACCOUNT_ID} --region ${AWS_REGION}
```
*Note:* The CodeArtifact authentication expires every 12 hours (default). You can move the step above to a GitLab workflow in case you want to automate the refresh of the token.

6. Fetch AWS CodeArtifact authorization token

The authorization token is required to interact with the repository. 

```bash
export CODEARTIFACT_AUTH_TOKEN=`aws codeartifact get-authorization-token --domain ${CODE_ARTIFACT_DOMAIN} --domain-owner ${AWS_ACCOUNT_ID} --region ${AWS_REGION} --query authorizationToken --output text`
```

### GitLab configuration

1. Setup the GitLab CLI

Install the [GitLab CLI](https://gitlab.com/gitlab-org/cli) version for your local workstation. See [official documentation](https://gitlab.com/gitlab-org/cli#installation): 

2. Run the following command to setup the GitLab host

```bash
glab auth login
```

You will get prompted for the GitLab instance details you want to log into, the GitLab hostname, and the API hostname. After entering those details, create a personal access from GitLab (https://<GITLAB URL>/-/profile/personal_access_tokens) and paste to the configuration input. 
Set the default git protocol to `SSH` and host API protocol to `HTTPS`. 

3. Create a GitLab repository

```bash
glab repo create ${GITLAB_REPO_NAME}
```




4. Create the environment variables required by Twine

```bash
glab variable set TWINE_PASSWORD --masked --value ${CODEARTIFACT_AUTH_TOKEN} --repo ${GITLAB_REPO_NAME}
glab variable set TWINE_USERNAME --masked --value ${CODE_ARTIFACT_DOMAIN} --repo ${GITLAB_REPO_NAME}
glab variable set AWS_ACCOUNT_ID --value ${AWS_ACCOUNT_ID} --repo ${GITLAB_REPO_NAME}
glab variable set DOMAIN_PREFIX --value ${DOMAIN_PREFIX} --repo ${GITLAB_REPO_NAME}
glab variable set CODE_ARTIFACT_DOMAIN --value ${CODE_ARTIFACT_DOMAIN} --repo ${GITLAB_REPO_NAME}
glab variable set CODE_ARTIFACT_REPO_NAME --value ${CODE_ARTIFACT_REPO_NAME} --repo ${GITLAB_REPO_NAME}
glab variable set AWS_REGION --value ${AWS_REGION} --repo ${GITLAB_REPO_NAME}
glab variable set PYTHON_MODULE_NAME --value ${PYTHON_MODULE_NAME} --repo ${GITLAB_REPO_NAME}
```

*Note:* Use `glab variable update` to set values for the variables if they already exist otherwise GitLab will return the error code 400 with message `<VARIABLE> has already been taken`

5. Upload the sample code to the repository

```bash
cp -r ./* ./cdk-python-module-gitlab-pipeline
cd ./cdk-python-module-gitlab-pipeline
git add . 
git commit -am "initial skeleton for cdk python module pipeline"
git push
```