# angorasix.core
Core project orchestrating all Angorasix services

# Build locally
For Maven projects:
`mvn clean package`
`npm run build` (not necessary - Docker builds for us)
`docker-compose -f docker-compose-local.yml build`

# Build for GCP
(gcp dockerfile uses base config only overriding image fields)
`docker-compose -f docker-compose-local.yml -f docker-compose.gcp.yml build <service>`

# Push to GCP Registry
`docker-compose -f docker-compose-local.yml -f docker-compose.gcp.yml push <service>`

# Debug container
`docker export gcp-debug-container > debug-container.tar`

# Push service config
Manually through console or 
`cd config/infra/prod-b`
`gcloud run services replace ` + <service>.yaml

### INFRA GCP

# CI/CD with Github Actions

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

# Google Cloud Run service-to-service communication

Receiveing service logs:

> "The request was not authorized to invoke this service. Read more at https://cloud.google.com/run/docs/securing/authenticating Additional troubleshooting documentation can be found at: https://cloud.google.com/run/docs/troubleshooting#401"

Resources: 
https://cloud.google.com/run/docs/authenticating/service-to-service

In general, we want to manage the request ourselves, so this resource is more suitable:
https://cloud.google.com/run/docs/securing/service-identity#identity-tokens