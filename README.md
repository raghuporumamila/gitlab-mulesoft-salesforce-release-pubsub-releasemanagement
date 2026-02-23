# Release Management: GitLab → MuleSoft → Salesforce → Pub/Sub on CloudHub

Here's a comprehensive walkthrough of the full release management pipeline.

---

## 1. Architecture Overview

The flow works like this:

**GitLab (Source Control & CI/CD)** → **MuleSoft Anypoint (Integration Layer)** → **Salesforce (Account Creation)** → **Google/Anypoint Pub/Sub (Event Publishing)** → **CloudHub (Runtime Deployment)**

---

## 2. GitLab Setup — Source Control & Pipeline

**Repository Structure**

Organize your MuleSoft project in GitLab like this:

```
/salesforce-account-integration
  /src/main/mule          ← Mule flows
  /src/main/resources     ← Properties, configs
  /src/test               ← MUnit tests
  pom.xml
  .gitlab-ci.yml          ← Pipeline definition
```

**Branch Strategy**

Use a trunk-based or Gitflow approach:
- `feature/*` → development branches
- `develop` → integration environment
- `release/*` → UAT/staging
- `main` → production

**`.gitlab-ci.yml` Pipeline**

```yaml
stages:
  - build
  - test
  - package
  - deploy-dev
  - deploy-uat
  - deploy-prod

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

build:
  stage: build
  script:
    - mvn clean compile -s settings.xml

test:
  stage: test
  script:
    - mvn test -s settings.xml
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml

package:
  stage: package
  script:
    - mvn package -DskipTests -s settings.xml
  artifacts:
    paths:
      - target/*.jar

deploy-dev:
  stage: deploy-dev
  script:
    - mvn deploy -DmuleDeploy
      -Dcloudhub.username=$CH_USERNAME
      -Dcloudhub.password=$CH_PASSWORD
      -Dcloudhub.environment=Development
      -Dcloudhub.workerType=MICRO
      -Dcloudhub.workers=1
      -Dapp.name=sf-account-integration-dev
  only:
    - develop

deploy-uat:
  stage: deploy-uat
  script:
    - mvn deploy -DmuleDeploy
      -Dcloudhub.environment=UAT
      -Dapp.name=sf-account-integration-uat
  only:
    - /^release\/.*/
  when: manual

deploy-prod:
  stage: deploy-prod
  script:
    - mvn deploy -DmuleDeploy
      -Dcloudhub.environment=Production
      -Dapp.name=sf-account-integration-prod
  only:
    - main
  when: manual
```

> Store `CH_USERNAME`, `CH_PASSWORD`, `SF_CLIENT_ID`, `SF_CLIENT_SECRET` etc. as **GitLab CI/CD Variables** (masked + protected).

---

## 3. MuleSoft Project — Salesforce Account Creation Flow

**`pom.xml` — Key Dependencies**

```xml
<dependencies>
  <!-- Salesforce Connector -->
  <dependency>
    <groupId>com.mulesoft.connectors</groupId>
    <artifactId>mule4-salesforce-connector</artifactId>
    <version>10.18.0</version>
  </dependency>

  <!-- Anypoint MQ / Pub Sub connector -->
  <dependency>
    <groupId>com.mulesoft.connectors</groupId>
    <artifactId>mule4-anypoint-mq-connector</artifactId>
    <version>4.0.6</version>
  </dependency>
</dependencies>

<!-- CloudHub Deployment Plugin -->
<plugin>
  <groupId>org.mule.tools.maven</groupId>
  <artifactId>mule-maven-plugin</artifactId>
  <version>3.8.7</version>
  <configuration>
    <cloudHubDeployment>
      <uri>https://anypoint.mulesoft.com</uri>
      <muleVersion>4.6.0</muleVersion>
      <username>${cloudhub.username}</username>
      <password>${cloudhub.password}</password>
      <applicationName>${app.name}</applicationName>
      <environment>${cloudhub.environment}</environment>
      <workerType>${cloudhub.workerType}</workerType>
      <workers>${cloudhub.workers}</workers>
      <region>us-east-1</region>
      <properties>
        <sf.clientId>${sf.clientId}</sf.clientId>
        <sf.clientSecret>${sf.clientSecret}</sf.clientSecret>
      </properties>
    </cloudHubDeployment>
  </configuration>
</plugin>
```

**Main Mule Flow — `salesforce-account-flow.xml`**

```xml
<flow name="create-salesforce-account-flow">

  <!-- 1. HTTP Listener — accepts account creation request -->
  <http:listener config-ref="HTTP_Listener_config" 
                 path="/api/accounts" 
                 allowedMethods="POST"/>

  <!-- 2. Validate & Transform payload -->
  <ee:transform doc:name="Map to SF Account">
    <ee:message>
      <ee:set-payload><![CDATA[
        %dw 2.0
        output application/java
        ---
        [{
          Name:         payload.accountName,
          Phone:        payload.phone,
          BillingCity:  payload.city,
          Industry:     payload.industry,
          Type:         "Prospect"
        }]
      ]]></ee:set-payload>
    </ee:message>
  </ee:transform>

  <!-- 3. Create Account in Salesforce -->
  <salesforce:create 
    config-ref="Salesforce_Config" 
    type="Account" 
    doc:name="Create SF Account"/>

  <!-- 4. Store Salesforce response -->
  <set-variable variableName="sfResponse" value="#[payload]"/>

  <!-- 5. Build Pub/Sub event payload -->
  <ee:transform doc:name="Build Event Payload">
    <ee:message>
      <ee:set-payload><![CDATA[
        %dw 2.0
        output application/json
        ---
        {
          eventType:   "ACCOUNT_CREATED",
          accountId:   vars.sfResponse[0].id,
          accountName: attributes.queryParams.accountName default "N/A",
          timestamp:   now() as String,
          status:      if (vars.sfResponse[0].success) "SUCCESS" else "FAILED",
          source:      "mulesoft-sf-integration"
        }
      ]]></ee:set-payload>
    </ee:message>
  </ee:transform>

  <!-- 6. Publish to Anypoint MQ (Pub/Sub) -->
  <anypoint-mq:publish 
    config-ref="Anypoint_MQ_Config"
    destination="salesforce-account-events"
    doc:name="Publish to MQ"/>

  <!-- 7. Return response to caller -->
  <ee:transform doc:name="Build HTTP Response">
    <ee:message>
      <ee:set-payload><![CDATA[
        %dw 2.0
        output application/json
        ---
        {
          message:   "Account created and event published successfully",
          accountId: vars.sfResponse[0].id,
          success:   vars.sfResponse[0].success
        }
      ]]></ee:set-payload>
    </ee:message>
  </ee:transform>

</flow>

<!-- Error Handling Flow -->
<error-handler name="Global_Error_Handler">
  <on-error-continue type="SALESFORCE:INVALID_INPUT">
    <ee:transform>
      <ee:message>
        <ee:set-payload><![CDATA[
          output application/json
          --- { error: "Invalid Salesforce input", details: error.description }
        ]]></ee:set-payload>
      </ee:message>
    </ee:transform>
    <http:response statusCode="400"/>
  </on-error-continue>

  <on-error-continue type="ANY">
    <logger message="Unhandled error: #[error.description]" level="ERROR"/>
    <http:response statusCode="500"/>
  </on-error-continue>
</error-handler>
```

---

## 4. Configuration Files — Environment-Specific Properties

Use environment-specific `.yaml` property files loaded at runtime:

**`config-dev.yaml`**
```yaml
sf:
  consumerKey: "${secure::sf.consumerKey}"
  consumerSecret: "${secure::sf.consumerSecret}"
  username: "dev-integration@yourorg.com"
  tokenEndpoint: "https://test.salesforce.com/services/oauth2/token"

anypoint:
  mq:
    clientId: "${secure::amq.clientId}"
    clientSecret: "${secure::amq.clientSecret}"
    url: "https://mq-us-east-1.anypoint.mulesoft.com/api/v1"
    destination: "salesforce-account-events-dev"

http:
  port: 8081
```

**`config-prod.yaml`**
```yaml
sf:
  tokenEndpoint: "https://login.salesforce.com/services/oauth2/token"
  destination: "salesforce-account-events-prod"
```

> Use **Mule Secure Configuration Properties** (`${secure::key}`) with encryption for secrets — never store plaintext credentials.

---

## 5. Anypoint MQ (Pub/Sub) Setup in CloudHub

**Step 1 — Create MQ Resources in Anypoint Platform**

1. Go to **Anypoint Platform → MQ**
2. Create a **Queue**: `salesforce-account-events-<env>`
3. Create an **Exchange** (for pub/sub fan-out): `account-events-exchange`
4. Bind the queue to the exchange
5. Generate **Client App credentials** per environment

**MQ Connector Config in Mule**

```xml
<anypoint-mq:config name="Anypoint_MQ_Config">
  <anypoint-mq:connection 
    url="${anypoint.mq.url}"
    clientId="${anypoint.mq.clientId}"
    clientSecret="${anypoint.mq.clientSecret}"/>
</anypoint-mq:config>
```

**Subscriber Flow (Consumer side)**

```xml
<flow name="consume-account-events-flow">
  <anypoint-mq:subscriber 
    config-ref="Anypoint_MQ_Config"
    destination="salesforce-account-events"
    doc:name="Subscribe to Account Events"/>

  <logger message="Received event: #[payload]" level="INFO"/>

  <!-- Route/process downstream based on event -->
  <choice>
    <when expression="#[payload.status == 'SUCCESS']">
      <!-- trigger downstream system, notification, etc. -->
    </when>
  </choice>
</flow>
```

---

## 6. CloudHub Deployment & Release Management

**Environments in Anypoint Platform**

Set up three environments mirroring your GitLab branches:

| GitLab Branch | Anypoint Environment | CloudHub App Name |
|---|---|---|
| `develop` | Development | `sf-account-integration-dev` |
| `release/*` | UAT | `sf-account-integration-uat` |
| `main` | Production | `sf-account-integration-prod` |

**Runtime Manager — Post-Deploy Config**

After deployment via the pipeline, verify in **Runtime Manager**:
- Worker size and count appropriate per env (MICRO for dev, SMALL/MEDIUM for prod)
- **Persistent Queues** enabled for production
- **Object Store v2** enabled
- Static IPs enabled for Salesforce IP whitelisting if needed

---

## 7. End-to-End Release Flow Summary

Here's the full release lifecycle:

```
Developer pushes to feature branch
        ↓
Merge Request raised in GitLab
        ↓
GitLab CI runs: build → unit tests (MUnit) → package
        ↓
Merge to develop → auto-deploy to CloudHub DEV
        ↓
QA validates in DEV → Salesforce sandbox account created, MQ event verified
        ↓
Create release/* branch → manual trigger deploys to UAT
        ↓
UAT sign-off → Merge Request to main
        ↓
Manual approval → deploy-prod stage fires
        ↓
CloudHub PROD app live → Salesforce Production accounts created → events flowing on MQ
```

---

## 8. Key Best Practices

**Security** — Use Anypoint Secrets Manager or the Mule Secure Properties Tool for all credentials. Never hardcode anything in flows or commit secrets to GitLab.

**Versioning** — Version your API specs in Anypoint Exchange first (RAML/OAS), then scaffold the Mule project from the spec. Tag GitLab releases with semantic versions matching your Exchange asset version.

**Rollback** — CloudHub supports redeployment of previous `.jar` artifacts. Keep your pipeline artifacts stored in GitLab for at least 30 days so you can redeploy a prior version manually if needed.

**Monitoring** — Use Anypoint Monitoring + Visualizer to trace the full flow from HTTP inbound → Salesforce → MQ. Set up alerts on error rates and MQ queue depth.

**MUnit Coverage** — Enforce a minimum coverage threshold (e.g., 80%) in your `pom.xml` so the pipeline fails if tests are insufficient before any deployment occurs.
