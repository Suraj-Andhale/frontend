include:
  - project: 'epay/devops/ci-templates'
    ref: main
    file: 'ci/build/vite.react.yml'
  - project: 'epay/devops/ci-templates'
    ref: main
    file: 'ci/sast/sast.yml'
  - project: 'epay/devops/ci-templates'
    ref: main
    file: 'ci/sast/fortify.sast.yml'
  - project: 'epay/devops/ci-templates'
    ref: main
    file: 'ci/release/version.yml'
  - project: 'epay/devops/ci-templates'
    ref: main
    file: 'ci/release/release.yml'
  - project: 'epay/devops/ci-templates'
    ref: main
    file: 'ci/build/podman.container.yml'
  - project: 'epay/devops/ci-templates'
    ref: main
    file: 'ci/deploy/deployment.yml'

stages:
- build
- test
- sast
- hp-fortify
- release
- dockerbuild
- deploy

variables:
  CI_TEMPLATE_REGISTRY_HOST: "registry.dev.sbiepay.sbi:8443"
  CI_TEMPLATE_REGISTRY_HOST_PREPROD: "epaynonprod-registry-quay-quay-enterprise.apps.preprod.epay.sbi"
  CI_TEMPLATE_REGISTRY_HOST_PROD: "registry.prod.epay.sbi:8443"
  CI_TEMPLATE_REGISTRY_HOST_PRODDR: "epayproddr-registry-quay-quay-enterprise.apps.dr.prod.epay.sbi"
  CI_TEMPLATE_REGISTRY_HOST_PRODDC: "proddc-registry-route-quay-enterprise.apps.dc.prod.epay.sbi"
  GIT_STRATEGY: clone
  CI_USERNAME: "gitlab-ci-token"
  RELEASE_NAME: null
  CHART_NAME: null
  IMAGE_NAME: "ubi9/$CHART_NAME"

.rules:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(main)|(develop)|(release)(\/.+)*$/'
    - if: '$CI_COMMIT_BRANCH =~ /^(main)|(develop)|(release)(\/.+)*$/'
    - when: never

# Development environment builds - used by dev, sit, uat
build-dev:
  variables:
    ENV: dev
  extends: 
    - .build
    - .rules

# Higher environment builds  
build-sit:
  variables:
    ENV: sit
  extends: 
    - .build
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

# Higher environment builds  
build-uat:
  variables:
    ENV: uat
  extends: 
    - .build
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

# Higher environment builds  
build-pre-prod:
  variables:
    ENV: pre-prod
  extends: 
    - .build
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

build-perf:
  variables:
    ENV: perf
  extends: 
    - .build
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

build-prod:
  variables:
    ENV: prod
  extends: 
    - .build
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(main)$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(main)$/
      when: manual
      allow_failure: true
    - when: never

sast:
  extends:
    - .sast
    - .rules

hp-fortify:
  variables:
    EXCLUDE_DIRS: "./**/dist/**/*"
    INCLUDE_DIRS: "./**/*.tsx,./**/*.ts,./**/*.js"
  extends:
    - .HPFortify
    - .rules

tag:
  extends:
    - .tag

# Docker build jobs for each environment
dockerbuild-dev:
  stage: dockerbuild
  variables:
    ENV: dev
  extends:
    - .dockerbuild
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(develop)$/'
    - if: '$CI_COMMIT_BRANCH =~ /^(develop)$/'
    - when: never

dockerbuild-sit:
  stage: dockerbuild
  variables:
    ENV: sit
    IMAGE_NAME: "ubi9/sit/$CHART_NAME"
  extends:
    - .dockerbuild
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

dockerbuild-uat:
  stage: dockerbuild
  variables:
    ENV: uat
    IMAGE_NAME: "ubi9/uat/$CHART_NAME"
  extends:
    - .dockerbuild
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

dockerbuild-pre-prod:
  stage: dockerbuild
  variables:
    ENV: pre-prod
    CI_TEMPLATE_REGISTRY_HOST: $CI_TEMPLATE_REGISTRY_HOST_PREPROD
    IMAGE_REGISTRY_USERNAME: $PREPROD_REGISTRY_USERNAME
    IMAGE_REGISTRY_PASS: $PREPROD_REGISTRY_PASSWORD
    IMAGE_NAME: "microservices/$CHART_NAME"
  extends:
    - .dockerbuild
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

dockerbuild-pre-prod-dc:
  stage: dockerbuild
  variables:
    ENV: pre-prod
    CI_TEMPLATE_REGISTRY_HOST: $CI_TEMPLATE_REGISTRY_HOST_PREPROD
    IMAGE_REGISTRY_USERNAME: $PREPROD_REGISTRY_USERNAME
    IMAGE_REGISTRY_PASS: $PREPROD_REGISTRY_PASSWORD
    IMAGE_NAME: "microservices/$CHART_NAME"
  extends:
    - .dockerbuild
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

dockerbuild-prod-dr:
  stage: dockerbuild
  variables:
    ENV: prod
    CI_TEMPLATE_REGISTRY_HOST: $CI_TEMPLATE_REGISTRY_HOST_PRODDR
    IMAGE_REGISTRY_USERNAME: $PROD_REGISTRY_USERNAME
    IMAGE_REGISTRY_PASS: $PROD_REGISTRY_PASSWORD
    IMAGE_NAME: "fe/$CHART_NAME"
  extends:
    - .dockerbuild
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(main)$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(main)$/
      when: manual
      allow_failure: true
    - when: never

dockerbuild-prod-dc:
  stage: dockerbuild
  variables:
    ENV: prod
    CI_TEMPLATE_REGISTRY_HOST: $CI_TEMPLATE_REGISTRY_HOST_PRODDC
    IMAGE_REGISTRY_USERNAME: $PROD_REGISTRY_USERNAME
    IMAGE_REGISTRY_PASS: $PROD_REGISTRY_PASSWORD
    IMAGE_NAME: "fe/$CHART_NAME"
  extends:
    - .dockerbuild
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(main)$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(main)$/
      when: manual
      allow_failure: true
    - when: never

# Deploy jobs with proper dependencies
deploy-dev:
  variables:
    ENV: dev
    SERVICE_NAME: $CI_PROJECT_NAME
  extends:
    - .trigger_cd
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(develop)$/
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(develop)$/
      when: manual
      allow_failure: true
    - when: never

deploy-sit:
  variables:
    ENV: sit
    SERVICE_NAME: $CI_PROJECT_NAME
  extends:
    - .trigger_cd
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

deploy-uat:
  variables:
    ENV: uat
    SERVICE_NAME: $CI_PROJECT_NAME
  extends:
    - .trigger_cd
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

deploy-pre-prod:
  variables:
    ENV: pre-prod
    SERVICE_NAME: $CI_PROJECT_NAME
    DEPLOYMENT_BRANCH: feature/pre-prod
  extends:
    - .trigger_cd
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: false
    - when: never

deploy-perf:
  variables:
    ENV: perf
    SERVICE_NAME: $CI_PROJECT_NAME
  extends:
    - .trigger_cd
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: false
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: false
    - when: never

deploy-pre-prod-dc:
  variables:
    ENV: pre-prod-dc
    SERVICE_NAME: $CI_PROJECT_NAME
  extends:
    - .trigger_cd
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(release)(\/.+)*$/
      when: manual
      allow_failure: true
    - when: never

deploy-prod-dr:
  variables:
    ENV: prod-dr
    SERVICE_NAME: $CI_PROJECT_NAME
    VERSION: $VERSION
  extends:
    - .trigger_cd
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(main)$/
      when: manual
      allow_failure: true
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(main)$/
      when: manual
      allow_failure: true
    - when: never

deploy-prod-dc:
  variables:
    ENV: prod-dc
    SERVICE_NAME: $CI_PROJECT_NAME
    VERSION: $VERSION
  extends:
    - .trigger_cd
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^(main)$/
      when: manual
      allow_failure: false
    - if: $CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ /^(main)$/
      when: manual
      allow_failure: true
    - when: never
