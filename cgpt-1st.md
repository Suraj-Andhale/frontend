# ===============================
# GitLab CI/CD for Frontend (React + Vite)
# ===============================

include:
  - project: "epay/devops/ci-templates"
    ref: main
    file:
      - "ci/build/vite.react.yml"
      - "ci/sast/sast.yml"
      - "ci/sast/fortify.sast.yml"
      - "ci/release/version.yml"
      - "ci/release/release.yml"
      - "ci/build/podman.container.yml"
      - "ci/deploy/deployment.yml"

stages:
  - build
  - test
  - sast
  - hp-fortify
  - release
  - dockerbuild
  - deploy

variables:
  GIT_STRATEGY: clone
  CI_USERNAME: gitlab-ci-token
  RELEASE_NAME: $CI_PROJECT_NAME
  CHART_NAME: $CI_PROJECT_NAME
  IMAGE_NAME: "ubi9/$CHART_NAME"

  # Registry Endpoints (from your infra)
  CI_TEMPLATE_REGISTRY_HOST: "registry.dev.sbiepay.sbi:8443"
  CI_TEMPLATE_REGISTRY_HOST_PREPROD: "epaynonprod-registry-quay-quay-enterprise.apps.preprod.epay.sbi"
  CI_TEMPLATE_REGISTRY_HOST_PROD: "registry.prod.epay.sbi:8443"

# ===============================
# BUILD STAGES
# ===============================

build-dev:
  variables:
    ENV: "dev"
    DOMAIN_URL: "https://dev-frontend.example.com"
  extends: [".build", ".rules"]
  script:
    - echo "Building React App for DEV"
    - npm ci
    - npm run build:dev
  artifacts:
    paths:
      - dist/
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

build-sit:
  variables:
    ENV: "sit"
    DOMAIN_URL: "https://sit-frontend.example.com"
  extends: [".build"]
  script:
    - echo "Building React App for SIT"
    - npm ci
    - npm run build:sit
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^release(\\/.*)*$/'

build-uat:
  variables:
    ENV: "uat"
    DOMAIN_URL: "https://uat-frontend.example.com"
  extends: [".build"]
  script:
    - echo "Building React App for UAT"
    - npm ci
    - npm run build:uat
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^release(\\/.*)*$/'

build-prod:
  variables:
    ENV: "prod"
    DOMAIN_URL: "https://frontend.example.com"
  extends: [".build"]
  script:
    - echo "Building React App for PROD"
    - npm ci
    - npm run build:prod
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ===============================
# DOCKER BUILD STAGES
# ===============================

dockerbuild-dev:
  stage: dockerbuild
  variables:
    ENV: "dev"
    IMAGE_NAME: "ubi9/dev/$CHART_NAME"
  extends: [".dockerbuild"]
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

dockerbuild-sit:
  stage: dockerbuild
  variables:
    ENV: "sit"
    IMAGE_NAME: "ubi9/sit/$CHART_NAME"
  extends: [".dockerbuild"]
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^release(\\/.*)*$/'

dockerbuild-uat:
  stage: dockerbuild
  variables:
    ENV: "uat"
    IMAGE_NAME: "ubi9/uat/$CHART_NAME"
  extends: [".dockerbuild"]
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^release(\\/.*)*$/'

dockerbuild-prod:
  stage: dockerbuild
  variables:
    ENV: "prod"
    IMAGE_NAME: "ubi9/prod/$CHART_NAME"
  extends: [".dockerbuild"]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ===============================
# DEPLOY STAGES (RHOCP + HELM)
# ===============================

deploy-dev:
  stage: deploy
  variables:
    ENV: "dev"
    SERVICE_NAME: "$CI_PROJECT_NAME"
  extends: [".trigger_cd"]
  script:
    - echo "Deploying DEV on OpenShift..."
    - oc login $OPENSHIFT_SERVER --token=$OPENSHIFT_TOKEN --insecure-skip-tls-verify
    - oc project frontend-dev
    - helm upgrade --install $RELEASE_NAME $CHART_NAME \
        --set image.repository=$CI_TEMPLATE_REGISTRY_HOST/$IMAGE_NAME \
        --set image.tag=dev \
        -f helm/values-dev.yaml
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-sit:
  stage: deploy
  variables:
    ENV: "sit"
    SERVICE_NAME: "$CI_PROJECT_NAME"
  extends: [".trigger_cd"]
  script:
    - echo "Deploying SIT on OpenShift..."
    - oc login $OPENSHIFT_SERVER --token=$OPENSHIFT_TOKEN --insecure-skip-tls-verify
    - oc project frontend-sit
    - helm upgrade --install $RELEASE_NAME $CHART_NAME \
        --set image.repository=$CI_TEMPLATE_REGISTRY_HOST/$IMAGE_NAME \
        --set image.tag=sit \
        -f helm/values-sit.yaml
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^release(\\/.*)*$/'

deploy-uat:
  stage: deploy
  variables:
    ENV: "uat"
    SERVICE_NAME: "$CI_PROJECT_NAME"
  extends: [".trigger_cd"]
  script:
    - echo "Deploying UAT on OpenShift..."
    - oc login $OPENSHIFT_SERVER --token=$OPENSHIFT_TOKEN --insecure-skip-tls-verify
    - oc project frontend-uat
    - helm upgrade --install $RELEASE_NAME $CHART_NAME \
        --set image.repository=$CI_TEMPLATE_REGISTRY_HOST/$IMAGE_NAME \
        --set image.tag=uat \
        -f helm/values-uat.yaml
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^release(\\/.*)*$/'

deploy-prod:
  stage: deploy
  variables:
    ENV: "prod"
    SERVICE_NAME: "$CI_PROJECT_NAME"
  extends: [".trigger_cd"]
  script:
    - echo "Deploying PROD on OpenShift..."
    - oc login $OPENSHIFT_SERVER --token=$OPENSHIFT_TOKEN --insecure-skip-tls-verify
    - oc project frontend-prod
    - helm upgrade --install $RELEASE_NAME $CHART_NAME \
        --set image.repository=$CI_TEMPLATE_REGISTRY_HOST_PROD/$IMAGE_NAME \
        --set image.tag=prod \
        -f helm/values-prod.yaml
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
