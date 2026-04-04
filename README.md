## Solar System NodeJS Application

A simple HTML+MongoDB+NodeJS project to display Solar System and it's planets.

---
## Requirements

For development, you will only need Node.js and NPM installed in your environement.

### Node
- #### Node installation on Windows

  Just go on [official Node.js website](https://nodejs.org/) and download the installer.
Also, be sure to have `git` available in your PATH, `npm` might need it (You can find git [here](https://git-scm.com/)).

- #### Node installation on Ubuntu

  You can install nodejs and npm easily with apt install, just run the following commands.

      $ sudo apt install nodejs
      $ sudo apt install npm

- #### Other Operating Systems
  You can find more information about the installation on the [official Node.js website](https://nodejs.org/) and the [official NPM website](https://npmjs.org/).

If the installation was successful, you should be able to run the following command.

    $ node --version
    v8.11.3

    $ npm --version
    6.1.0

---
## Install Dependencies from `package.json`
    $ npm install

## Run Unit Testing
    $ npm test

## Run Code Coverage
    $ npm run coverage

## Run Application
    $ npm start

## Access Application on Browser
    http://localhost:3000/

# Application Logic w.r.t Kubernetes
                +-----------------------------+
                |   User's Browser / Client   |
                +-----------------------------+
                             |
                             v
                +-----------------------------+
                |        Ingress (NGINX)      |
                | solar-system-<ns>.<ip>.nip.io|
                |   TLS termination (HTTPS)   |
                +-----------------------------+
                             |
                             v
                +-----------------------------+
                |   Service: solar-system     |
                |   Type: NodePort (3000)     |
                |   Selector: app=solar-system|
                +-----------------------------+
                             |
                +-----------------------------+
                |   Pods (Deployment)         |
                |   solar-system containers   |
                |   Express app on port 3000  |
                |   Env from mongo-db-creds   |
                +-----------------------------+
                             |
                             v
                +-----------------------------+
                |   MongoDB Atlas Cluster     |
                |   supercluster.d83jj...     |
                |   Database: superData       |
                +-----------------------------+

## CD - Azure setup
  #### Azure service principle through GUI
   ```
   -- Microsoft Entra ID -> select App registrations > New registration > Create
   -- Generate a Client Secret   , Copy client id and secret
   -- Assign Permissions (RBAC) -- at Resource Group Level / Subscription Level -- Role: Select Contributor (
   Create the Service Principal
```
  ####  service principle through CLI
  ```
  az ad sp create-for-rbac --name "github-actions-sp" --role contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID> \
  --sdk-auth
```
  AZURE_CREDENTIALS secret  with values
  ```
  {
  "clientId": "<GUID>",
  "clientSecret": "<GUID>",
  "subscriptionId": "<GUID>",
  "tenantId": "<GUID>"
}
```
#### The easiest way to get this JSON is by running the following command in the Azure CLI (or Azure Cloud Shell)
```
az ad sp create-for-rbac --name "myApp-github-actions" --role contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID> \
  --sdk-auth
```
#### assign a role-based access control (RBAC) role to your Azure App Registration (Service Principal), you can use the Azure Portal or the Azure CLI.Method 

1: Azure PortalNavigate to the Resource: Go to the resource where you want to grant access (e.g., your AKS Cluster, Resource Group, or Subscription).Open Access Control: Select Access control (IAM) from the left-hand menu.Add Assignment: Click Add > Add role assignment.Select Role: Choose a role that fits your needs.

Common roles for AKS include:
 - Azure Kubernetes Service Cluster User Role: To download kubeconfig.
 - Azure Kubernetes Service RBAC Admin: For full administrative control.
 - Contributor: To manage almost all resources.
 - Assign to Principal: Your App Registration name, and select it.Finalise: Click Review + assign to complete the process.
 
2: Azure CLIUse the az role assignment create command. You will need your App's Client ID (App ID) or Object ID.bash# Get the resource ID of your cluster
```
AKS_ID=$(az aks show --resource-group <YourResourceGroup> --name <YourAKSCluster> --query id -o tsv)
```
#### Assign the role (example: RBAC Admin)
```
az role assignment create \
  --assignee <Your-App-Client-ID> \
  --role "Azure Kubernetes Service RBAC Admin" \
  --scope $AKS_ID
```
Use code with caution.Key ConsiderationsPropagation Delay: New role assignments can take up to 5 minutes (and sometimes longer for service principals) to fully propagate through the system.Principle of Least Privilege: Always start with the most restrictive role needed for the task to maintain high security.Scope: You can restrict access to a specific namespace by adding /namespaces/<namespace-name> to the end of your scope ID.Are you setting this up for a CI/CD pipeline like GitHub Actions
