[logo]: https://res.cloudinary.com/snyk/image/upload/v1533761770/logo-1_wtob68.svg
![Snyk Security Scanning](https://res.cloudinary.com/snyk/image/upload/v1533761770/logo-1_wtob68.svg)
# Snyk Codefresh Example
This example application has a sample application along with a Codefresh pipeline that can build, scan, and promote a Docker image. 

*Warning* These instructions are incomplete. Some variables in the pipeline need to be updated to match your environment. Update coming soon. 
## Using the plugin
### Scan Code
``` 
RunningUnitTests:
    stage: scan
    title: Running Unit Tests
    image: '${{BuildingDockerImage}}'
    working_directory: IMAGE_WORK_DIR
    entry_point:
      - /bin/sh
      - /codefresh/volume/cf-generated/unit_test_script
    create_file:
      path: /codefresh/volume/cf-generated
      name: unit_test_script
      content: |-
        npm install -g snyk
        snyk test || true
    on_success:
      metadata:
        set:
          - '${{BuildingDockerImage.imageId}}':
              - CF_QUALITY: true
    on_fail:
      metadata:
        set:
          - '${{BuildingDockerImage.imageId}}':
              - CF_QUALITY: false
```

### Scan Docker Image
```
  SnykScanImage:
      stage: scan
      type: composition
      composition:
        version: '2'
        services:
          targetimage:
            image: ${{BuildingDockerImage}} # Must be the Docker build step name
            command: sh -c "exit 0"
            labels:
              build.image.id: ${{CF_BUILD_ID}} # Provides a lookup for the composition
      composition_candidates:
        scan_service:
          image: aarlaudsnyk/snyk-container-scan-docker
          command: python snyk-cli.py "${{IMAGE_NAME}}:${{CF_BRANCH_TAG_NORMALIZED}}"
          environment:
          - SNYK_TOKEN=${{SNYK_TOKEN}}
          - SNYK_ORG=${{SNYK_ORG}}
          depends_on:
            - targetimage
          volumes: # Volumes required to run DIND
            - /var/run/docker.sock:/var/run/docker.sock
            - /var/lib/docker:/var/lib/docker
      add_flow_volume_to_composition: true
      on_success: # Execute only once the step succeeded
        metadata: # Declare the metadata attribute
          set: # Specify the set operation
            - ${{BuildingDockerImage.imageId}}: # Select any number of target images
              - SECURITY_SCAN: true

      on_fail: # Execute only once the step failed
        metadata: # Declare the metadata attribute
          set: # Specify the set operation
            - ${{BuildingDockerImage.imageId}}: # Select any number of target images
              - SECURITY_SCAN: false 
```

## Instructions

### Pre-requisites 
- [Codefresh account](https://codefresh.io/) (free or paid)
- [Snyk account](https://snyk.io/) (free or paid)
- Dockerhub account (Optional)

### Add Repo to Codefresh
Signin to Codefresh and click "Add Repository" from the [repositories screen](https://g.codefresh.io/repositories). Paste in the url for this repo and click next. Then select "I have a Codefresh.yml" and put `./.codefresh/codefresh.yml` for the path. This will preview the Codefresh yaml, then follow the instructions to finish creating the pipeline.

### Add Environmental Variables 
You can type in the variables by hand, or just copy and paste the following:
```
PORT=8080
SNYK_ORG=aarlaud-snyk-demo
IMAGE_NAME=aarlaudsnyk/trainingapp
SNYK_TOKEN=addapikeyhere
```

Select "Import from Text" to import. 

We'll also add a token from Snyk. You can get this from your [Snyk account settings](https://app.snyk.io/account). Add this variable with `SNYK_TOKEN` as the key. Then check encrypt to store the token securely. 

### Add Dockerhub (optional)
Codefresh has a built-in private Docker registry. In this example we're building and pushing a public image so we'll use Docker hub. Follow the instructions in the [Docker Registry integration page](https://g.codefresh.io/account-conf/integration/registry).

You can skip this step by removing the promote to Dockerhub step. 

### Go run your pipelne. 