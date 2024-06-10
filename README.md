
# DevSecOps Project with GitLab

This project demonstrates a DevSecOps pipeline using GitLab CI/CD. The pipeline includes various stages for SAST, DAST, building Docker images, and deploying applications. This README provides instructions for setting up the necessary tools, configuring GitLab variables, and understanding the pipeline.

## Prerequisites

- Docker
- GitLab Runner
- SonarQube
- DefectDojo

## Installation Steps

### GitLab

1. **Install GitLab:**
   - Follow the [GitLab installation guide](https://about.gitlab.com/install/) for your operating system.

2. **Set up a GitLab Runner:**
   - Follow the [GitLab Runner installation guide](https://docs.gitlab.com/runner/install/) and register the runner with your GitLab instance.

### SonarQube

1. **Install SonarQube:**
   - Pull the SonarQube Docker image:
     ```sh
     docker pull sonarqube
     ```
   - Run the SonarQube container:
     ```sh
     docker run -d --name sonarqube -p 9000:9000 sonarqube
     ```

2. **Generate SonarQube Token:**
   - Access SonarQube at `http://localhost:9000`.
   - Log in with the default credentials (`admin`/`admin`).
   - Navigate to **My Account** > **Security**.
   - Generate a new token and save it.

### DefectDojo

1. **Install DefectDojo:**
   - Pull the DefectDojo Docker image:
     ```sh
     docker pull defectdojo/defectdojo-django
     ```
   - Run the DefectDojo container:
     ```sh
     docker run -d --name defectdojo -p 8080:8080 defectdojo/defectdojo-django
     ```

2. **Generate DefectDojo API Key:**
   - Access DefectDojo at `http://localhost:8080`.
   - Log in with the default credentials (`admin`/`admin`).
   - Navigate to **Configuration** > **API v2** > **API Key**.
   - Generate a new API key and save it.

## Configuring GitLab Variables

In your GitLab project, navigate to **Settings** > **CI/CD** > **Variables** and add the following variables:

- `NVD_API_KEY`: Your NVD API key for Dependency Check.
- `SONAR_HOST_URL`: The URL of your SonarQube server (e.g., `http://localhost:9000`).
- `SONAR_TOKEN`: Your SonarQube token.
- `SSH_PRIVATE_KEY`: Your SSH private key for deployment.
- `DOJO_IMPORT_SCAN_URL`: The URL of your DefectDojo import scan endpoint (e.g., `http://localhost:8080/api/v2/import-scan/`).
- `DD_API_KEY`: Your DefectDojo API key.

## Understanding the `.gitlab-ci.yml` File

The `.gitlab-ci.yml` file defines the CI/CD pipeline for this project. Here's an explanation of each stage and job:

```yaml
stages:
  - sast_scans
  - build
  - trivy_docker_scan
  - deploy
  - dast_scans
  - defectdojo

dependency_check_scan:
  stage: sast_scans
  image: owasp/dependency-check:latest
  variables:
    NVD_API_KEY: $NVD_API_KEY
  script:
    - |
      if [ -d "./dependency-check-data" ]; then
        echo "Using cached Dependency-Check data"
      else
        echo "No cache found, downloading Dependency-Check data"
      fi
      /usr/share/dependency-check/bin/dependency-check.sh --project "Dependency-Check Project" --scan ./ --format "XML" --out dependency-check-report.xml --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}
  cache:
    key: dependency-check-data
    paths:
      - dependency-check-data
  artifacts:
    paths:
      - dependency-check-report.xml
  allow_failure: true
  only:
    - main
  tags:
    - default

sonarqube_scan:
  stage: sast_scans
  image: ubuntu:latest
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
    SONAR_PROJECT_KEY: "Netflix"
    SONAR_HOST_URL: $SONAR_HOST_URL
    SONAR_TOKEN: $SONAR_TOKEN
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  before_script:
    - apt-get update && apt-get install -y npm
    - npm install -g sonar-report
  script:
    - |
      sonar-scanner \
      -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
      -Dsonar.sources=. \
      -Dsonar.host.url=${SONAR_HOST_URL} \
      -Dsonar.login=${SONAR_TOKEN}
  artifacts:
    paths:
      - sonarreport.html
  allow_failure: true
  only:
    - main
  tags:
    - default

trivy_fs_scan:
  stage: sast_scans
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - |
      trivy fs --format json -o trivyfs-report.json .
  artifacts:
    paths:
      - trivyfs-report.json
  allow_failure: true
  only:
    - main
  tags:
    - default

build_docker_image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  script:
    - docker build -t myapp:${CI_COMMIT_SHORT_SHA} .
    - docker save myapp:${CI_COMMIT_SHORT_SHA} | gzip > myapp_${CI_COMMIT_SHORT_SHA}.tar.gz
  artifacts:
    paths:
      - myapp_${CI_COMMIT_SHORT_SHA}.tar.gz
  tags:
    - default

trivy_docker_scan:
  stage: trivy_docker_scan
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  script:
    - apk add --no-cache curl
    - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
    - gunzip -c myapp_${CI_COMMIT_SHORT_SHA}.tar.gz | docker load
    - trivy image --format json -o trivy-docker-report.json myapp:${CI_COMMIT_SHORT_SHA}
  artifacts:
    paths:
      - trivy-docker-report.json
  dependencies:
    - build_docker_image
  allow_failure: true
  only:
    - main
  tags:
    - default

deploy_local:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh
  script:
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan -H 192.168.126.129 >> ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
    - scp myapp_${CI_COMMIT_SHORT_SHA}.tar.gz pentism@192.168.126.129:/tmp/
    - ssh -o StrictHostKeyChecking=no pentism@192.168.126.129 "docker load < /tmp/myapp_${CI_COMMIT_SHORT_SHA}.tar.gz && docker run -d -p 8180:80 --name myapp_container myapp:${CI_COMMIT_SHORT_SHA}"
  variables:
    SSH_PRIVATE_KEY: $SSH_PRIVATE_KEY
  tags:
    - default
  dependencies:
    - build_docker_image

owasp_zap_scan:
  stage: dast_scans
  image: ghcr.io/zaproxy/zaproxy:stable
  script:
    - mkdir /zap/wrk
    - zap-full-scan.py -t http://192.168.126.129:8180 -r zap-report.html
    - cp /zap/wrk/zap-report.html ./zap-report.html
  artifacts:
    paths:
      - zap-report.html
  dependencies:
    - deploy_local
  allow_failure: true
  only:
    - main
  tags:
    - default
```

### Explanation of the `.gitlab-ci.yml` File

- **Stages**: Define the sequence of stages in the pipeline.
  - `sast_scans`: Includes SAST tools like Dependency Check, SonarQube, and Trivy filesystem scan.
  - `build`: Builds the Docker image.
  - `trivy_docker_scan`: Scans the built Docker image.
  - `deploy`: Deploys the application.
  - `dast_scans`: Includes DAST tools like OWASP ZAP.

- **Jobs**: Define the tasks in each stage.
  - `dependency_check_scan`: Runs Dependency Check.
  - `sonarqube_scan`: Runs SonarQube scan.
  - `trivy_fs_scan`: Runs Trivy filesystem scan.
  - `build

_docker_image`: Builds the Docker image.
  - `trivy_docker_scan`: Scans the Docker image using Trivy.
  - `deploy_local`: Deploys the Docker image to a local server.
  - `owasp_zap_scan`: Runs OWASP ZAP scan.

### Setting Up GitLab CI/CD

1. **Create a new GitLab project**.
2. **Add your `.gitlab-ci.yml` file to the root of your repository**.
3. **Commit and push the changes**.
4. **Monitor the pipeline** in the GitLab CI/CD section.

## Conclusion

This setup provides a comprehensive CI/CD pipeline for a DevSecOps project, ensuring code quality and security throughout the development lifecycle.
