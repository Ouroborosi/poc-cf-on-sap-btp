# SAP BTP Cloud Foundry POC
Forked from [gs-reactive-rest-service](https://github.com/spring-guides/gs-reactive-rest-service).

[English](README.md)

## 概要
開始之前請確認已申請 SAP trial account，且 SAP Cloud Foundry Runtime 環境已設定完成。本試驗計劃將一 `hello-world` API，透過 GitHub Actions 與 cf Cli 部署至 SAP Cloud Foundry Runtime。

## 前置需求
1. SAP Trail Account with Cloud Foundry Runtime setup.
2. JDK 17
3. Maven
4. Cloud Foundry CLI (on self-hosted runner)

## 發佈

### 手動發佈 (使用 CI/CD 工具前)
尚未採用 CI/CD 工具前，我們可以透過 cfCli 指令來做部署。

**cf api**
設定與目標 Cloud Foundry 實例進行 API 通訊。

> 💡**如何找到 Cloud Foundry 實例 API 端點 URL？**
> 可於 SAP BTP Cloud Foundry Environment 區塊找到
> 
```shell
cf api <api_endpoint>
```

**cf login**
登錄 Cloud Foundry 實例。

```shell
cf login

# > API endpoint: https://xxxx.xxxx.xxxx
# > 
# > Email:: <SAP trial account email>
# > password: <SPA trial account password>
```

**Build Project**
將應用程式打包為`jar`檔。

```shell
mvn clena pcakge
```

**cf push**
將應用程式部署到 Cloud Foundry。
`-i`: 設定建立的實體個數
`-m`: 服務宣告使用的記憶體大小
`-p`: 部署檔案的路徑
`-b`: 指定 buildpack
```shell
cf push hello-app-2 -i 1 -m 256M -p target/reactive-rest-service-complete-0.0.1-SNAPSHOT.jar -b sap_java_buildpack
```

### 整合 Github Actions
透過[自定義 Github Action 的工作流](https://github.com/Ouroborosi/cf-cli-action)於 push master branch 時觸發建置作業，並於建置完成後將 cf cli manifest.yml 檔案內的 path 
屬性替換為建置後產生的部署檔路徑。

於 environment secret 內設定下列參數:
- `CF_API`
- `CF_USER`
- `CF_PASSWORD`
- `CF_ORG` (位於 in the SAP BTP Cloud Foundry 竟設定區塊內的 ORG Name)
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