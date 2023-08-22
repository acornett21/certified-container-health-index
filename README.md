# Red Hat Container Certification Health Index Grade Walk Through

## By: Adam D. Cornett

## Overview
The Red Hat Container Certification lets you build, certify, and distribute your containerized application. In order to meet the security and support requirements from enterprise customers, the container testing and validation is broken into two main parts:

- The `preflight` tool, which does static analysis of the container to ensure the container conforms to the
  [certification policies](https://access.redhat.com/documentation/en-us/red_hat_software_certification/8.66/html/red_hat_openshift_software_certification_policy_guide/assembly-requirements-for-container-images_openshift-sw-cert-policy-introduction).
- CVE Vulnerability Scanning, which checks for vulnerabilities in Red Hat content within the container and gives the image a grade.

Since this process consists of two parts, and the CVE scanning happens in an async fashion, we will walk through the steps to retrieve the grade of the image.

## Prerequisites
- Register as a technology partner [here](http://connect.redhat.com/login) [in order to complete certification] if not already a partner.
- Review the container certification workflow [documentation](https://access.redhat.com/documentation/en-us/red_hat_software_certification/8.65/html/red_hat_software_certification_workflow_guide/assembly_working-with-containers_configuring-the-system-and-running-tests-by-using-cockpit-for-non-containerized-application)
- A Container application [project](https://access.redhat.com/documentation/en-us/red_hat_software_certification/8.65/html/red_hat_software_certification_workflow_guide/proc_creating-a-container-application-project_openshift-sw-cert-workflow-working-with-containers) within the Partner Connect Portal
- A container tool ie Podman/Docker
- Latest release of Preflight Certification tool
  - Binary can be downloaded [here](https://github.com/redhat-openshift-ecosystem/openshift-preflight/releases/latest)
  - Container can be pulled from `podman pull quay.io/opedev/preflight:stable`
- A CI system that can make `curl` request (OpenShift Pipelines, Github Actions, Jenkins, etc)

## Build and Certify Your Application Container
To follow along with the below steps, all the prerequisites should have been completed.

### Pre-step: Export Environment Variables
Below are some environment variables that will be used across multiple steps.
> **_NOTE:_** Documentation on how to obtain an Pyxis API Token or Certification Project ID can be found in the [Red Hat Software Certification Workflow Guide](https://access.redhat.com/documentation/en-us/red_hat_software_certification/8.66/html/red_hat_software_certification_workflow_guide/proc_running-the-certification-test-suite_openshift-sw-cert-workflow-complete-pre-certification-checklist-for-containers).

```bash
export PYXIS_API_TOKEN=abcdefghijklmnopqrstuvwxyz123456
export IMAGE_TAG=registry.example.org/your-namespace/your-image:sometag
export CERTIFICATION_PROJECT_ID=1234567890a987654321bcde
```

### Step 1: Building the Container
```bash 
podman login registry.example.org -u=user --authfile=./temp-auth.json
podman build -t $(IMAGE_TAG) . && podman push $(IMAGE_TAG) --authfile=./temp-auth.json
```

### Step 2: Running Preflight Certification tool
```bash
preflight check container registry.example.org/your-namespace/your-image:sometag \
--submit \
--pyxis-api-token=$(PYXIS_API_TOKEN) \
--certification-project-id=$(CERTIFICATION_PROJECT_ID) \
--docker-config=./temp-auth.json
```

## Querying Red Hat API for Health Index
After the image is submitted by `preflight`, an async process kicks off to grade the image. Since the grade won't be returned instantly, we will have to poll for it. In the below script we wait 5 seconds between calls, but your workflow may want to poll at some other interval, or in some other fashion that fits your use case better.

### Step 1: Call Red Hat API Until a Grade is Returned
> **_NOTE:_** Below we use `skopeo inspect` instead of `podman inspect` due a [bug](https://github.com/containers/podman/issues/3761) in podman's `Digest` field.

```bash
export CONTAINER_SHA=$(skopeo inspect docker://registry.example.org/your-namespace/your-image:sometag | jq '.["Digest"]')

grade=""
until [$grade != ""]
do
  echo "checking for Health Index Grade"
  grade="$(curl -s -X 'GET' 'https://catalog.redhat.com/api/containers/v1/images?filter=docker_image_digest=='"${CONTAINER_SHA}"'&page_size=100&page=0' \
    -H 'accept: application/json' \
    -H 'X-API-KEY: '"${PYXIS_API_TOKEN}"'' | jq -r '{id: .data[0]._id, freshness_grades: .data[0].freshness_grades[] | {grade, creation_date}}')"
  sleep 5
done
echo "Health Index Grade: $grade"
```

## Getting Help
Any issues related specifically to the certification steps can be directed to our
[Partner Acceleration Desk (PAD)](https://access.redhat.com/articles/6463941).

## Further Reading
If you are interested in exploring what other API's Red Hat Certification has to offer, please check out our public [API Specifications](https://catalog.redhat.com/api/containers/docs/).

If you are interesting learning how you could automate Container Certification using OpenShift Pipelines and Quay, please check out this related [article](https://connect.redhat.com/blog/using-openshift-pipelines-cicd-and-quay-container-certification)
