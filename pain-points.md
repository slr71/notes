# DE Deployment Pain Points

- Generating the SSL keys for kubernetes is automated, but getting the required data into the Ansible inventories is
  not.
- A lot of information about our deployment process is coded in Jenkins, which forces the information to be duplicated
  for external deployments.
- Since some deployments are dependent on others, it would be nice to have a deployment order available. It's not
  strictly necessary, but it would be nice.
- Not all of the services are required if some features aren't being used, and there's no automated way to deploy only
  the services that are required.
- The accepted keys and signing keys for terrain are all hard-coded into the deployment, which forces the deployment
  template to be updated for every DE deployment. It would be nice if we changed this so that the template doesn't have
  to be different for every deployment.
- The apps service doesn't currently handle the case where the user has no permissions for any apps very well. An
  `unable to get app categories` error appears instead.
- There isn't currently a good way to grant permissions to the default list of public apps in the DE. I had to add the
  permissions mostly manually.
