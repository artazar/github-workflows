This repository contains generic GitHub Workflow and GitHub Actions templates that can be used as "Reusable workflows".

An example of a calling repository that invokes some of the stored workflows:

```
name: Java Maven Build & Deploy

on:
  push:
    branches:
      - develop
      - main
      - stage
    paths-ignore:
      - '.github/**'
  pull_request:
    branches:
      - develop
    paths-ignore:
      - '.github/**'
  workflow_dispatch:

jobs:
  check:
    if: github.event_name == 'pull_request'
    uses: artazar/github-workflows/.github/workflows/build_maven_check.yml@main
    secrets: inherit

  integration:
    if: github.event_name == 'pull_request'
    uses: artazar/github-workflows/.github/workflows/build_maven_test_integration.yml@main
    secrets: inherit

  build:
    if: github.event_name != 'pull_request'
    uses: artazar/github-workflows/.github/workflows/build_maven_publish.yml@main
    with:
      tomcat_image: ghcr.io/artazar/utils/tomcat:main
    secrets: inherit

  vars:
    if:
      contains('
        refs/heads/develop
        refs/heads/main
        refs/heads/stage
        ', github.ref)
    runs-on: ubuntu-latest
    outputs:
      namespace: ${{ steps.ns.outputs.namespace }}
    steps:
    - name: Map branch to namespace
      id: ns
      run: |   
        if [ "${GITHUB_REF}" = 'refs/heads/main' ]
        then
          NAMESPACE="myapp-prod"
          CLUSTER="k8s-001"
        elif [ "${GITHUB_REF}" = 'refs/heads/develop' ]
        then
          NAMESPACE="myapp-dev"
          CLUSTER="k8s-001"
        elif [ "${GITHUB_REF}" = 'refs/heads/stage' ]
        then 
          NAMESPACE="myapp-stg"
          CLUSTER="k8s-001"
        fi

        echo ::set-output name=cluster::${CLUSTER}
        echo "Cluster is set to ${CLUSTER}"

        echo ::set-output name=namespace::${NAMESPACE}
        echo "Namespace is set to ${NAMESPACE}"

  deploy:
    if:
      contains('
        refs/heads/develop
        refs/heads/main
        refs/heads/stage
        ', github.ref)
    uses: artazar/github-workflows/.github/workflows/deploy_kubernetes_flux.yml@main
    needs: [build, vars]
    with:
      app_name: ${{ github.event.repository.name }}
      app_version: ${{ needs.build.outputs.app_version }}
      namespace: ${{ needs.vars.outputs.namespace }}
      cluster: ${{ needs.vars.outputs.cluster }}
      flux_repo: k8s-flux
    secrets: inherit
```

The example is applicable for a Java/Maven project that runs PR code style checks, builds a Sprint Boot microservice app and deploys to a specified namespace in a Kubernetes cluster.