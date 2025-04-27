# Dev Settings

## IDE - IntelliJ:
Set up code style settings in IntelliJ IDE:
`Preferences > Editor > Code Style > Kotlin > (Gear Icon) > Impor Scheme > IntelliJ IDEA code style > (search for infra/resources/dev-settings/kotlin XML file`

Then setup Ktlint Plugin (in `distract free` mode):
https://pinterest.github.io/ktlint/latest/install/setup/ 

## Maven - format:
`mvn clean verify`

- just ktlint checks:
`mvn antrun:run@ktlint`

- ktlin format:
`mvn antrun:run@ktlint-format`


# DevOps

## Build locally: Option A - Full build from Docker (takes more time)
`docker-compose -f docker-compose-local.yml -f docker-compose-local.full-build.yml up -d`

## Build locally: Option B - Faster Docker Build (uses pre-build resources)
For Maven projects:
`mvn clean package`
`npm run build` (not necessary - Docker builds for us)
`docker-compose -f docker-compose-local.yml build`

## Build for GCP
(gcp dockerfile uses base config only overriding image fields)
`docker-compose -f docker-compose-local.yml -f docker-compose.gcp.yml build <service>`

## Push to GCP Registry
`docker-compose -f docker-compose-local.yml -f docker-compose.gcp.yml push <service>`

## Debug container
`docker export gcp-debug-container > debug-container.tar`

## Push service config
Manually through console or 
`cd config/infra/prod-b`
`gcloud run services replace ` + <service>.yaml

# INFRA GCP

## Tutorial: GCP 01 - Deploy NEW service
https://www.iorad.com/player/2354117/GCP-01---Deploy-NEW-service

## CI/CD with Github Actions

1- Set up identity federation accounts:
https://cloud.google.com/blog/products/identity-security/secure-your-use-of-third-party-tools-with-identity-federation

2- Part of the previous process, but to add additional Github repos to permissions once the setup is ready:

```
gcloud iam service-accounts add-iam-policy-binding service-account@angorasix.iam.gserviceaccount.com \
--role="roles/iam.workloadIdentityUser" \
--member="principal://iam.googleapis.com/projects/{project-id-#}/locations/global/workloadIdentityPools/GitHub-actions-pool/subject/repo:AngoraSix/{service-name}:ref:refs/heads/main"
```

3- Create Github setup (clone from other projects - using base actions defined in this repo):
https://cloud.google.com/blog/products/devops-sre/deploy-to-cloud-run-with-github-actions

## Google Cloud Run service-to-service communication

Receiveing service logs:

> "The request was not authorized to invoke this service. Read more at https://cloud.google.com/run/docs/securing/authenticating Additional troubleshooting documentation can be found at: https://cloud.google.com/run/docs/troubleshooting#401"

Resources: 
https://cloud.google.com/run/docs/authenticating/service-to-service

In general, we want to manage the request ourselves, so this resource is more suitable:
https://cloud.google.com/run/docs/securing/service-identity#identity-tokens

## Manage Git pushes:
Tags with the format `vMajor.Minor.Patch` (major.minor.patch) version trigger a new artifact/image push.

To create a new tag in the current commit:
```
git tag -a vMajor.Minor.Patch -m "New release purpose"
git push origin --tags
```

Remember to protect the `v*.*.*` tags in the Github repository configs.

If we need to re-push a tag (mainly because the CI plan failed, and we need to fix the codebase and re-run it):
```
git tag --delete vMajor.Minor.Patch
git push origin :vMajor.Minor.Patch
```
(remove the tag locally, and delete it in the remote repo)

### Clean git tag:
`git tag -d v0.0.0`

`git push --delete origin v0.0.0`

## Setup new Cloud Run container - Backend service:
1- Git push version tag to get container created (the deploy will fail though)

2- Go to the GCP Console for Cloud Run

3- In the list, select the container, and then select on "Permissions". Add Principal: "Default Service Compute Account" and "a6-services-id@angorasix.iam.gserviceaccount.com" as in other services, with Cloud Run Invoker. Save.

4- Go into container details and `Edit & Deploy New Version`:
4.A- Variables and Secrets: 
- A6_*_PORT to 8080
- A6_*_MONGO_DB_NAME to correspnding name
- A6_*_MONGO_DB_PARAMS to `?retryWrites=true&w=majority`
- A6_*_OAUTH_SECURITY_ISSUER_URI to corresponding value (check other services)
- (SECRET) A6_*_MONGO_DB_URI reference to corresponding secret
- any other variables and secrets for the service
4.B- Networking tab:
- Selct `Connect to a VPC for outbound traffic` and use `Use Serverless VPC Access connectors` selecting the corresponding a6-vpc entry.
- Select `Route all traffic to the VPC`
4.C- Deploy
4.D- Copy the URL from the info dashboard, to be used in the Gateway Service setup

5- Deploy new version of Gateway using git tag push, if suitable (most likely yes since we need to route to the new service)

5.A- Go to the Gateway container details and `Edit & Deploy New Version`:
setting up the new "A6_GATEWAY_*_URI" variable to the URL we copied before.

## Setup new Cloud Run container - Frontend service:
1- Git push version tag to get container created (the deploy will fail though)

2- Go to the GCP Console for Cloud Run

3- In the list, select the container, and then select on "Permissions". Add Principal: "allUsers" as in other front-end services, with Cloud Run Invoker. Save.

4- Go into container details and `Edit & Deploy New Version`:
4.A- Container Port:
- 80
- and Healthcheck to port 80 as well
4.A- Variables and Secrets: 
- NEXTAUTH_URL to base HTTPS domain
- NEXTAUTH_URL_INTERNAL to container internal URL (in container details of step 4)
- *_APP_API_SERVER_BASE_URL to internal Gateway service URL (from GatewaySvc container details)
- *_PUBLIC_APP_API_BROWSER_BASE_URL to base HTTPS domain
- *_APP_OAUTH_PROVIDER_ISSUER to Contributors internal URL (from Contributors container details)
- *_APP_OAUTH_PROVIDER_TOKEN_ENDPOINT to Contributors internal URL + /oauth2/token
- *_APP_OAUTH_PROVIDER_AUTHORIZATION_ENDPOINT to Contributors internal URL + /oauth2/authorize
- *_APP_OAUTH_PROVIDER_JWKS_ENDPOINT to Contributors internal URL + /oauth2/jwks
- *_APP_OAUTH_CLIENT_ID to matching client id configured in Contributors
- *_APP_OAUTH_REDIRECT_URI to base public HTTPS domain URL + /api/auth/callback/angorasixspring 
- *_APP_OAUTH_FW_DEBUG to false
- *_APP_INFRA_GOOGLE_CLOUDRUN_AUTH_ENABLED to true
- *_PUBLIC_APP_THIRDPARTIES_GOOGLEANALYTICS_ID to corresponding ID
- *_PUBLIC_APP_THIRDPARTIES_GOOGLEANALYTICS_ID (After registering new property in GA)
- *_PUBLIC_APP_THIRDPARTIES_GOOGLERECAPTCHA_ID (After adding to valid URLs ing GCP)

SECRETS
- *_APP_MAIN_SECRET (Create new value)
- *_APP_OAUTH_CLIENT_SECRET to existing secret
- *_APP_OAUTH_JWT_SECRET (create new value)
- *_APP_THIRDPARTIES_GOOGLERECAPTCHA_SECRET to existing secret
- any other variables and secrets for the service
4.B- Networking tab:
- Selct `Connect to a VPC for outbound traffic` and use `Use Serverless VPC Access connectors` selecting the corresponding a6-vpc entry.
- Select `Route all traffic to the VPC`
4.C- Security:
Service account to `a6-services-id@angorasix.iam.gserviceaccount.com`
4.C- Deploy

5- Go to Secret Manager in GCP
- for new Secrets go into details and the Permissions
- add `a6-services-id` with role `Secret Manager Secret Accessor`

6- re-deploy

7- If want to enable same LOGIN, add RedirectURI in Contributors service for new env property
- Push changes in Contributor Service, and add corresponding env property

# Infra Publish Maven Artifacts

## Create and Distribute GPG Token:
https://central.sonatype.org/publish/requirements/gpg/#signing-a-file

Can use `openssl rand -base64 32` to generate passphrase / secrets

## ASCII-armor key (create private key)
https://dzone.com/articles/how-to-publish-artifacts-to-maven-central

gpg --armor --export-secret-keys YOUR_KEY_ID > private.gpg

## Update Maven Variables
Passphrase and Private Key

## (Re)trigger the build

# Infra RabbitMQ:

## Tutorial: RabbitMQ 01 - How to set up a remote instance using CloudAMQP.
https://www.iorad.com/player/2354096/RabbitMQ-01---How-to-set-up-a-remote-instance-using-CloudAMQP-

## Tutorial: RabbitMQ 02 - Get and set host and credentials
https://www.iorad.com/player/2354101/RabbitMQ-02---Get-and-set-host-and-credentials

# Config Decisions:

## Port ranges:

* 100[00-49]: Bare Infra
    * Gateway: 10000
    * Media: 10001
    * Integrations.Iframe: 10002

* 101[00-99]: Domain Infra
    * Contributors: 10100
    * Events: 10101
    * Notifications: 10102

* 102[00-99]: Projects
    * Core: 10200
    * Presentation: 10201

* 103[00-99]: Project Managements
    * Core: 10300
    * Tasks: 10301
    * Integrations: 10302
    * Accounts: 10303

* 107[00-99]: Fronts
    * AngoraSix: 10700
    * AindaNow: 10701
    * Cooperativemos: 10702

* 109[00-49]: Others
    * Clubs: 10900
    * Messaging: 10901

* 109[50-99]: Others - Isolated
    * Integrations-Iframe: 10950
    * Surveys: 10951

## Local Postman
### Collection:
`https://angora-s-six.postman.co/workspace/AngoraSix~d1d6924a-6c95-4f3c-8012-b231bc5bb898/collection/32020175-92414228-aaa4-4a0b-93f5-b930368e9f1f?action=share&creator=32020175&active-environment=32020175-d1a1dea9-4f02-482b-b651-51308e57c00e`

### Environment:
`https://angora-s-six.postman.co/workspace/AngoraSix~d1d6924a-6c95-4f3c-8012-b231bc5bb898/environment/32020175-d1a1dea9-4f02-482b-b651-51308e57c00e?action=share&creator=32020175&active-environment=32020175-d1a1dea9-4f02-482b-b651-51308e57c00e`