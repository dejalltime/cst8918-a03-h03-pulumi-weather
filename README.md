CST8918 - DevOps: Infrastructure as Code  
Prof: Robert McKenney

# LAB-A03 Pulumi Weather App

In this hands-on lab activity you will revisit the weather app from [LAB-A01](https://github.com/rlmckenney/cst8918-w25-a01-weather). This time you have been asked to make the solution more robust and further reduce redundant hits on the OpenWeather API. To accomplish this, the team has decided to replace the in-app memory cache with a Redis cache that will be shared by all container instances.

Additionally, it is time to deploy this app to our public cloud provider (Azure). The team has decided to use [Pulumi](https://pulumi.com) to manage the provisioning of infrastructure resources and deployment of the application.

## High-level system overview

This is a simplified diagram of the system architecture that you will be building and deploying.

![Azure Container App system diagram](./system-architecture.png)

## Objectives

### Part One (Lab-A03)

- Use Pulumi to provision the IaC resources
  - Azure Container Registry (ACR)
  - Azure Container Instances (ACI)
- Use Pulumi to package the application into a Docker container and deploy it

### Part Two (Hybrid-H03)

- Add proper secret handling for the OpenWeather API key
- Modify the application code to utilize a shared Redis instance
- Update the Pulumi config to provision additional IaC resources
  - Azure Cache for Redis
- Update the container image version and redeploy with Pulumi

### Teams

- Work in pairs using a shared GitHub repo.
- Divide the tasks and work in parallel.
- Practice committing, pushing and resolving conflicts (if needed).
- The submitted repo must include meaningful commits from each team member.
- Only one team member needs to submit on Brightspace.

### Starter Repo

> [!IMPORTANT]
> Do not simply continue from the previous assignment's repo.

Fork and then clone this repo to have a clean common starting point. There are a couple of changes from the original repo, most significantly the Dockerfile. It would be worth your time to look at the structure of the Dockerfile. It is a good example of how to define a multi-stage build for a node.js app.

Create a working branch for this lab called `lab-a03`. Do all of your work in this branch.

**Do not push code directly to the `main` branch.**

> [!TIP]
> Remember to run `npm install` in the project folder after you clone it. This will install all of the Node.js dependencies required to do local testing with the Remix dev server.

## Part One (A01) - Add Pulumi to the project

Create a new folder at the top level of the project called `infrastructure`, then make that your working directory.

```sh
mkdir infrastructure && cd infrastructure
```

Use the Pulumi CLI to initialize this infrastructure folder as a Pulumi project that will use Typescript.

```sh
pulumi new typescript
```

Provide the following values when prompted:

- _project name:_ cst8918-a03-infra
- _project description:_ A Remix app deployed with the Azure Container App service
- _stack name:_ prod

This will create a minimal Pulumi config for a **prod** stack. It doesn't do anything yet. Let's fix that!

### Set the _prod_ environment config variables

There are going to be a few environment variables that we need to set. That can be done either via the Pulumi CLI, or by directly editing the `Pulumi.prod.yaml` file (which does not exist until you create it or add the first config value).

Start with setting the desired Azure region for your production deployment. Our organization is using `westus3`. We are going to use the `@pulumi\azure-native` SDK to interface with Azure, so the command is ...

```sh
pulumi config set azure-native:location westus3
```

We will use the Pulumi Docker-Build library module to generate the containerize image. It needs to know the path to find the _Dockerfile_ for our application, the public port number to expose, and the CPU and Memory resource limits.

> [!TIP]
> See the [Pulumi Docker-Build Library Docs](https://www.pulumi.com/registry/packages/docker-build/) for more information.

You can edit the `Pulumi.prod.yaml` file directly to add the remaining config params. It should look like this.

```yaml
config:
  azure-native:location: westus3
  cst8918-a03-infra:appPath: ../
  cst8918-a03-infra:containerPort: '80'
  cst8918-a03-infra:publicPort: '80'
  cst8918-a03-infra:cpu: '1'
  cst8918-a03-infra:memory: '2'
  cst8918-a03-infra:prefixName: 'cst8918-a03-<your-username>'
  cst8918-a03-infra:imageTag: 'v0.1.0'
```

> [!IMPORTANT]
> Please update the `prefixName` value to replace `<your-username>` with your correct college username. e.g. my username is `mckennr`, so my prefixName would be `cst8918-a03-mckennr`. We will use this prefixName in several places when creating various infrastructure resources.

### Install some helper modules

Since we are going to deploy Docker containers on Azure, you will need to install a couple of extra Pulumi modules. Make sure that you are still in the `infrastructure` folder, then run ...

```sh
npm i @pulumi/docker-build @pulumi/azure-native
```

> [!TIP]
> There are many other plugin modules available for Pulumi. You can find them and read their documentation on the [Pulumi Registry](https://www.pulumi.com/registry/). Spend some time browsing them and learning about the different modules that are available.

### Declare the desired infrastructure

Its time to define the desired infrastructure shape. Open up the `index.ts` file in the `infrastructure` folder. Right now, it should just have one line, importing Pulumi.

```ts
import * as pulumi from '@pulumi/pulumi'
```

#### Load the application config information

To begin, load the configuration variables for the given environment (stack)

> [!NOTE]
> Use the `pulumi config` command to view and set the values of the configuration options. The code below uses the `require` method to throw an error if the config value is not set. See the [Pulumi Config Docs](https://www.pulumi.com/docs/reference/pkg/nodejs/pulumi/pulumi/classes/Config.html) for more information.

```ts
// Import the configuration settings for the current stack.
const config = new pulumi.Config()
const appPath = config.require('appPath')
const prefixName = config.require('prefixName')
const imageName = prefixName
const imageTag = config.require('imageTag')
// Azure container instances (ACI) service does not yet support port mapping
// so, the containerPort and publicPort must be the same
const containerPort = config.requireNumber('containerPort')
const publicPort = config.requireNumber('publicPort')
const cpu = config.requireNumber('cpu')
const memory = config.requireNumber('memory')
```

#### Define the container registry

According to the architecture diagram, our solution calls for a private container image repository rather than using Docker Hub as we did with the initial prototype. It is a best practice to place the application solution resources in a resource group for easier identification and monitoring.

Create a new resource group. Use the prefixName + '-rg' for the name.
At the top of your index.ts file, import the `resources` and `containerregistry` modules from the `@pulumi/azure-native` package.

```ts
import * as resources from '@pulumi/azure-native/resources'
import * as containerregistry from '@pulumi/azure-native/containerregistry'
```

Append this definition code to the bottom of your index.ts file. **We will use the cost-optimized _basic_ registry SKU.**

```ts
// Create a resource group.
const resourceGroup = new resources.ResourceGroup(`${prefixName}-rg`)

// Create the container registry.
const registry = new containerregistry.Registry(`${prefixName}ACR`, {
  resourceGroupName: resourceGroup.name,
  adminUserEnabled: true,
  sku: {
    name: containerregistry.SkuName.Basic,
  },
})
```

Before you can tell the Docker module to store the container image in the container registry, you will need to get the registry's authentication credentials.

```ts
// Get the authentication credentials for the container registry.
const registryCredentials = containerregistry
  .listRegistryCredentialsOutput({
    resourceGroupName: resourceGroup.name,
    registryName: registry.name,
  })
  .apply((creds) => {
    return {
      username: creds.username!,
      password: creds.passwords![0].value!,
    }
  })
```

This seems like a good time to check our work. _Temporarily,_ append these stack output instructions so that you can make sure that everything is working up to this point.

```ts
export const acrServer = registry.loginServer
export const acrUsername = registryCredentials.username
```

And then run it in the terminal ...

```sh
pulumi up
```

Review the `plan` output, and correct any errors if needed. After you say 'yes' to apply the update, you should see the something similar to the following near the end of the output ...

```sh
Outputs:
  + acrServer  : "containerregistry84d73e7d.azurecr.io"
  + acrUsername: "containerRegistry84d73e7d"
```

> [!NOTE]
> Pulumi automatically adds the random characters to the end of the registry name to ensure uniqueness. Yours will be slightly different.

**SUCCESS !**

> [!TIP]
> OK now you can delete those last two `export` lines. You won't need them any more.

#### Create the Docker image and store it in the container registry

Import the `@pulumi/docker-build` module at the top of the index.ts file and then append the container definition to the bottom of the file. Of note, the `platforms` option tells Docker to build the image for both `linux/amd64` and `linux/arm64` runtime architectures. This will make sure to pull the right base image when processing the Dockerfile, and give us the flexibility to deploy the same Docker image to both platforms.

```ts
// Other imports at the top of the module
import * as dockerBuild from '@pulumi/docker-build'

// ... rest of the code

// Define the container image for the service.
const image = new dockerBuild.Image(`${prefixName}-image`, {
  tags: [pulumi.interpolate`${registry.loginServer}/${imageName}:${imageTag}`],
  context: { location: appPath },
  dockerfile: { location: `${appPath}/Dockerfile` },
  target: 'production',
  platforms: ['linux/amd64', 'linux/arm64'],
  push: true,
  registries: [
    {
      address: registry.loginServer,
      username: registryCredentials.username,
      password: registryCredentials.password,
    },
  ],
})
```

Notice the code above references the `imageTag` variable. You can use this to assign the current version of the application before publishing it. This makes it really easy to roll-back if needed! You should set it now. Use the pulumi CLI to set it to `v0.2.0` -- our app is still in the prototype stage :wink:

```sh
pulumi config set imageTag "v0.2.0"
```

#### Create an Azure Container App service container group

Create a container group in the Azure Container App service and make it publicly accessible. Our system design calls for a linux container host (Azure also supports Windows hosts). This is a big chunk of code. Let's break it down.

- the first section defines the container group meta info, including the host OS type and the image registry to pull from
- then it defines the container images to use, any environment variables to inject, which target port to use on the container and resource limits for the containers.
- the last section defines the public ingress: DNS name and IP address.

```ts
// Other imports at the top of the module
import * as containerinstance from '@pulumi/azure-native/containerinstance'

// ... rest of the code

// Create a container group in the Azure Container App service and make it publicly accessible.
const containerGroup = new containerinstance.ContainerGroup(
  `${prefixName}-container-group`,
  {
    resourceGroupName: resourceGroup.name,
    osType: 'linux',
    restartPolicy: 'always',
    imageRegistryCredentials: [
      {
        server: registry.loginServer,
        username: registryCredentials.username,
        password: registryCredentials.password,
      },
    ],
    containers: [
      {
        name: imageName,
        image: image.ref,
        ports: [
          {
            port: containerPort,
            protocol: 'tcp',
          },
        ],
        environmentVariables: [
          {
            name: 'PORT',
            value: containerPort.toString(),
          },
          {
            name: 'WEATHER_API_KEY',
            value: '<your-secret-key>',
          },
        ],
        resources: {
          requests: {
            cpu: cpu,
            memoryInGB: memory,
          },
        },
      },
    ],
    ipAddress: {
      type: containerinstance.ContainerGroupIpAddressType.Public,
      dnsNameLabel: `${imageName}`,
      ports: [
        {
          port: publicPort,
          protocol: 'tcp',
        },
      ],
    },
  },
)
```

> [!WARNING]
> Replace **\<your-secret-key\>** with your real API key. Yes, unencrypted for now -- we will take care of that in part two.

##### Define the output values

You will need to know the final IP address and the public URL to test the app in your browser.

```ts
// Export the service's IP address, hostname, and fully-qualified URL.
export const hostname = containerGroup.ipAddress.apply((addr) => addr!.fqdn!)
export const ip = containerGroup.ipAddress.apply((addr) => addr!.ip!)
export const url = containerGroup.ipAddress.apply(
  (addr) => `http://${addr!.fqdn!}:${containerPort}`,
)
```

##### Test it!

We still need to handle the secret encryption, but before we go any further it is a good idea to test your deployment.

```sh
pulumi up
```

After the deployment completes, you should be able to see the deployed resources in your Azure portal, and you should be able to open the output `URL` (e.g. http://cst8918-a03-mckennr.westus3.azurecontainer.io/) in your browser to see the app running.

### Demo / Submit

**Only one partner submit for the group on Brightspace**

When you have completed Part One, make sure that you have committed all of your changes with git, and pushed your commits up to GitHub. Remember, this should be on a branch call `lab-a03`. Also add a screenshot of your browser showing the application running -- make sure the public URL is clearly visible. The screenshot should be in the root of your project folder with the name `lab-a03.png`.

Submit a link to your GitHub repo for this assignment in Brightspace.

## Clean-up!

When you are all done, don't forget to clean up the unneeded Azure resources.

```sh
pulumi destroy
```

> [!CAUTION]
> Failing to do this may exceed your Azure subscription limit, resulting in academic penalties!

## Next Steps

The instructions for Part Two of this practice scenario are in the [README-h03.md](https://github.com/rlmckenney/cst8918-w24-a03-weather-pulumi/blob/main/README-h03.md) file in this repo.
