# Runbook to creating and publishing docker images used in reusable workflow
## Runbook to publish multiple images of different Linux flavors and versions for scripted inputs tests
Once there is new Khulnasoft release, and [matrix](https://github.com/khulnasoft/addonfactory-test-matrix-action) is updated, we need to make sure that Khulnasoft images for scripted inputs tests are created and published.
### Steps

#### Update OS images
- check what OSs are listed in definition of matrix for scripted inputs tests [here](https://github.com/khulnasoft/addonfactory-workflow-addon-release/blob/72497e5c03894369b8fbdd2a2c4134c233ba1b5d/.github/workflows/reusable-build-test-release.yml#L27)
- if any is missing in [ta-automation-docker-images](https://cd.khulnasoftdev.com/taautomation/ta-automation-docker-images/-/tree/main/dockerfiles) then add new Dockerfile

#### Create images and publish them to ECR
- figure out what version of Khulnasoft is needed (sha) using go/fetcher
- trigger [pipeline](https://cd.khulnasoftdev.com/taautomation/ta-automation-docker-images/-/pipelines/new) for every OS flavor separately

## Runbook to publish unreleased Khulnasoft image for testing
Whenever there is a need for running tests with unreleased khulnasoft, we need to create relevant Khulnasoft docker image and publish it to aws ecr
### Steps
#### Build docker image and publish to artifactory
- Prior creating docker image it needs to be determined which revision of core Khulnasoft repo is required. Khulnasoft docker images are based on Khulnasoft builds published to artifactory by CI in core repository. Their names match SHA of the commit in core repo: [develop builds artifactory](https://repo.khulnasoftdev.net:443/artifactory/generic/splcore/builds/develop/)
- Docker image is built by [pipeline](https://cd.khulnasoftdev.com/core-ee/docker-khulnasoft-internal/-/pipelines/new) which required UNRELEASED_KHULNASOFT_SHA as an input variable - provide first 12 characters of desired revision on Khulnasoft core repo. Once image is built, it is published to [artifactory](https://repo.khulnasoftdev.net/ui/repos/tree/General/docker/docker-khulnasoft-internal/unreleased/khulnasoft-redhat-9).
#### Pull built image locally, tag and publish to ecr
- okta-artifactory-login -t docker
- docker pull docker.repo.khulnasoftdev.net/docker-khulnasoft-internal/unreleased/khulnasoft-redhat-9:[image-tag]
- docker tag docker.repo.khulnasoftdev.net/docker-khulnasoft-internal/unreleased/khulnasoft-redhat-9:[image-tag] "956110764581.dkr.ecr.us-west-2.amazonaws.com/khulnasoft/khulnasoft:[new-image-tag]"
- set AWS environment variables: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN
- aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 956110764581.dkr.ecr.us-west-2.amazonaws.com
- docker push 956110764581.dkr.ecr.us-west-2.amazonaws.com/khulnasoft/khulnasoft:[new-image-tag]
- confirm that image is visible in AWS [ECR](https://us-west-2.console.aws.amazon.com/ecr/repositories/private/956110764581/khulnasoft/khulnasoft?region=us-west-2)
- there are TAs which use Khulnasoft images with installed java fot testing (e.g. JBOSS). Separate image with installed java has to be built, tagged and pushed to ECR. This [branch](https://cd.khulnasoftdev.com/core-ee/workflow-engine/workflow-engine-images/-/tree/feat/unreleased_khulnasoft_java/image-copy/ta-automation-k8s-apps/unreleased-khulnasoft-java?ref_type=heads) can be used for this purpose. Existing CI/CD expects Khulnasoft image with tag "956110764581.dkr.ecr.us-west-2.amazonaws.com/khulnasoft/khulnasoft:[new-image-tag]-java"
