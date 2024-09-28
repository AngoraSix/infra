# Dev Settings

## IDE - IntelliJ:
Set up code style settings in IntelliJ IDE:
`Preferences > Editor > Code Style > Kotlin > (Gear Icon) > Impor Scheme > IntelliJ IDEA code style > (search for infra/resources/dev-settings/kotlin XML file`

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

* 107[00-99]: Fronts
    * AngoraSix: 10700
    * AindaNow: 10701

* 109[00-99]: Others
    * Clubs: 10900
