# SAP BTP Cloud Foundry POC
Forked from [gs-reactive-rest-service](https://github.com/spring-guides/gs-reactive-rest-service).

[中文](README_zh.md)

## Summary
Before starting, please make sure you have applied for an SAP trial account and have completed the SAP Cloud Foundry Runtime setup. This exercise will deploy a `hello-world` API to SAP Cloud Foundry Runtime using GitHub Actions and cf Cli.

## Prerequisites
1. SAP Trail Account with Cloud Foundry Runtime setup.
2. JDK 17
3. Maven
4. Cloud Foundry CLI (on self-hosted runner)

## Deployment

### Before CI/CD (the manual way)
Before adopting CI/CD tools, we can deploy using cfCli commands in a manual way.

**cf api**
Set up and communicate with the target Cloud Foundry instance through API.

> 💡**How to find the API endpoint URL of a Cloud Foundry instance?**
> Can be found in the SAP BTP Cloud Foundry Environment section.
>
```shell
cf api <api_endpoint>
```

**cf login**
Log in to the Cloud Foundry instance

```shell
cf login

# > API endpoint: https://xxxx.xxxx.xxxx
# > 
# > Email:: <SAP trial account email>
# > password: <SPA trial account password>
```

**Build Project**
Package application to `jar` file.

```shell
mvn clena pcakge
```

**cf push**
Deploy an application to Cloud Foundry.
`-i`: Set the number of instances to create.
`-m`: Set the memory limit for the app.
`-p`: Path to the app file to deploy.
`-b`: Specify the buildpack to be used.

```shell
cf push <app-name> -i 1 -m 256M -p <path-to-app-files> -b sap_java_buildpack
```

### Integrate with Github Actions
Set up a [customized workflow](https://github.com/Ouroborosi/cf-cli-action) in Github Actions to trigger a build job on push to the `master` 
branch, and 
after the build is completed, replace the `path` property in the cf CLI `manifest.yml` file with the deployment file path generated from the build.

Set the following parameters in the environment secret:
- `CF_API`
- `CF_USER`
- `CF_PASSWORD`
- `CF_ORG` (The ORG Name located in the in SAP BTP Cloud Foundry Environment section)
- `CF_SPACE`

```yaml
name: CF CLI Action

on:
  push:
    branches:
      - master

jobs:
  build:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      - run: ./mvnw clean package
  deploy:
    runs-on: self-hosted
    needs: build
    environment: trial
    steps:
      - run: |
          sed -i "s/\$ARTIFACT_NAME/$(find ./target -maxdepth 1 -type f -name "*.jar" | grep -o '[^/]*\.jar$')/" manifest.yml
      - uses: Ouroborosi/cf-cli-action@master
        with:
          cf_api: ${{ secrets.CF_API }}
          cf_username: ${{ secrets.CF_USER }}
          cf_password: ${{ secrets.CF_PASSWORD }}
          cf_org: ${{ secrets.CF_ORG }}
          cf_space: ${{ secrets.CF_SPACE }}
          command: push -f manifest.yml.bak
```