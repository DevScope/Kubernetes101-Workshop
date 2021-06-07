# Dotnet Tye Intro Hands-On-Lab

This lab is composed by 3 exercises:

**Content**

- [Dotnet Tye Intro Hands-On-Lab](#dotnet-tye-intro-hands-on-lab)
- [Exercise 1: Running .NET applications locally using Tye in an integrated environment](#exercise-1-running-net-applications-locally-using-tye-in-an-integrated-environment)
  - [Frontend Backend sample with tye run](#frontend-backend-sample-with-tye-run)
    - [Running a single application with tye run](#running-a-single-application-with-tye-run)
    - [Running multiple applications with tye run](#running-multiple-applications-with-tye-run)
    - [Getting the frontend to communicate with the backend](#getting-the-frontend-to-communicate-with-the-backend)
    - [Next Steps](#next-steps)
    - [Troubleshooting](#troubleshooting)
      - [Certificate is invalid exception on Linux](#certificate-is-invalid-exception-on-linux)
        - [Generate the certificate](#generate-the-certificate)
- [Exercise 2: Deploying applications to a Kubernetes cluster quickly and easily](#exercise-2-deploying-applications-to-a-kubernetes-cluster-quickly-and-easily)
  - [Getting Started with Deployment](#getting-started-with-deployment)
    - [Deploying the application](#deploying-the-application)
    - [Exploring tye.yaml](#exploring-tyeyaml)
    - [Undeploying the application](#undeploying-the-application)
    - [Next Steps](#next-steps-1)
- [Exercise 3: Adding external services needed by the system and deploying them to a Kubernetes cluster](#exercise-3-adding-external-services-needed-by-the-system-and-deploying-them-to-a-kubernetes-cluster)
  - [Adding redis to an application](#adding-redis-to-an-application)
    - [Deploying redis](#deploying-redis)



Tye is yet a beta project, but with great capabilities and features that will allow it to grow its popularity faster among developers.

In the `hello-tye` folder there's three exercises using Tye, plus a folder called `docs` which is some documentation powered by Microsoft.

Tye project github: https://github.com/dotnet/tye

# Exercise 1: Running .NET applications locally using Tye in an integrated environment
## Frontend Backend sample with tye run

This tutorial will demonstrate how to use [`tye run`](https://github.com/dotnet/tye/blob/main/docs/reference/commandline/tye-run.md) to run a multi-project application. If you haven't so already, follow the [Getting Started Instructions](https://github.com/dotnet/tye/blob/main/docs/getting_started.md) to install tye.

### Running a single application with tye run

1. Make a new folder called `microservice` and navigate to it:

    ```text
    mkdir microservice
    cd microservice
    ```

1. Create a frontend project:

    ```text
    dotnet new razor -n frontend
    ```

1. Run this new project with `tye` command line:

    ```text
    tye run frontend
    ```

    With just a single application, tye will do two things: start the frontend application and run a dashboard. Navigate to <http://localhost:8000> to see the dashboard running.

    The dashboard should show the `frontend` application running.

    - The `Logs` column has a link to view the streaming logs for the service.
    - the `Bindings` column has links to the listening URLs of the service.
    
    Navigate to the `frontend` service using one of the urls on the dashboard in the *Bindings* column. It should be in the form of <http://localhost:[port]> or <https://localhost:[port]>.

    The dashboard will use port 8000 if possible. Services written using ASP.NET Core will have their listening ports assigned randomly if not explicitly configured.

### Running multiple applications with tye run

1. If you haven't already, stop the existing `tye run` command using `Ctrl + C`. Create a backend API that the frontend will call inside of the `microservices/` folder.

    ```text
    dotnet new webapi -n backend
    ```

1. Create a solution file and add both projects

    ```text
    dotnet new sln
    dotnet sln add frontend backend
    ```

    You should have a solution called `microservice.sln` that references the `frontend` and `backend` projects.

2. Run the `tye` command line in the folder with the solution.

    ```text
    tye run
    ```

    The dashboard should show both the `frontend` and `backend` services. You can navigate to both of them through either the dashboard of the url outputted by `tye run`.

    > :warning: The `backend` service in this example was created using the `webapi` project template and will return an HTTP 404 for its root URL.

### Getting the frontend to communicate with the backend

Now that we have two applications running, let's make them communicate. By default, `tye` enables service discovery by injecting environment variables with a specific naming convention. For more information on, see [service discovery](https://github.com/dotnet/tye/blob/main/docs/reference/service_discovery.md).

1. If you haven't already, stop the existing `tye run` command using `Ctrl + C`. Open the solution in your editor of choice.

2. Add a file `WeatherForecast.cs` to the `frontend` project.

    ```C#
    using System;

    namespace frontend
    {
        public class WeatherForecast
        {
            public DateTime Date { get; set; }

            public int TemperatureC { get; set; }

            public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);

            public string Summary { get; set; }
        }
    }
    ```

    This will match the backend `WeatherForecast.cs`.

3. Add a file `WeatherClient.cs` to the `frontend` project with the following contents:

   ```C#
    using System.Net.Http;
    using System.Text.Json;
    using System.Threading.Tasks;

    namespace frontend
    {
        public class WeatherClient
        {
            private readonly JsonSerializerOptions options = new JsonSerializerOptions()
            {
                PropertyNameCaseInsensitive = true,
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            };
    
            private readonly HttpClient client;
    
            public WeatherClient(HttpClient client)
            {
                this.client = client;
            }
    
            public async Task<WeatherForecast[]> GetWeatherAsync()
            {
                var responseMessage = await this.client.GetAsync("/weatherforecast");
                var stream = await responseMessage.Content.ReadAsStreamAsync();
                return await JsonSerializer.DeserializeAsync<WeatherForecast[]>(stream, options);
            }
        }
    }
   ```

4. Add a reference to the `Microsoft.Tye.Extensions.Configuration` package to the frontend project

    ```txt
    dotnet add frontend/frontend.csproj package Microsoft.Tye.Extensions.Configuration  --version "0.7.0-*"
    ```

5. Now register this client in `frontend` by adding the following to the existing `ConfigureServices` method to the existing `Startup.cs` file:

   ```C#
   ...
   public void ConfigureServices(IServiceCollection services)
   {
       services.AddRazorPages();
        /** Add the following to wire the client to the backend **/
       services.AddHttpClient<WeatherClient>(client =>
       {
            client.BaseAddress = Configuration.GetServiceUri("backend");
       });
       /** End added code **/
   }
   ...
   ```

   This will wire up the `WeatherClient` to use the correct URL for the `backend` service.

6. Add a `Forecasts` property to the `Index` page model under `Pages\Index.cshtml.cs` in the `frontend` project.

    ```C#
    ...
    public WeatherForecast[] Forecasts { get; set; }
    ...
    ```

   Change the `OnGet` method to take the `WeatherClient` to call the `backend` service and store the result in the `Forecasts` property:

   ```C#
   ...
   public async Task OnGet([FromServices]WeatherClient client)
   {
        Forecasts = await client.GetWeatherAsync();
   }
   ...
   ```

7. Change the `Index.cshtml` razor view to render the `Forecasts` property in the razor page:

   ```cshtml
   @page
   @model IndexModel
   @{
        ViewData["Title"] = "Home page";
    }

   <div class="text-center">
       <h1 class="display-4">Welcome</h1>
       <p>Learn about <a href="https://docs.microsoft.com/aspnet/core">building Web apps with ASP.NET Core</a>.</p>
   </div>

   Weather Forecast:

    <table class="table">
        <thead>
            <tr>
                <th>Date</th>
                <th>Temp. (C)</th>
                <th>Temp. (F)</th>
                <th>Summary</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var forecast in @Model.Forecasts)
            {
                <tr>
                    <td>@forecast.Date.ToShortDateString()</td>
                    <td>@forecast.TemperatureC</td>
                    <td>@forecast.TemperatureF</td>
                    <td>@forecast.Summary</td>
                </tr>
            }
        </tbody>
    </table>
   ```

8.  Run the project with [`tye run`](https://github.com/dotnet/tye/blob/main/docs/reference/commandline/tye-run.md) and the `frontend` service should be able to successfully call the `backend` service!

    When you visit the `frontend` service you should see a table of weather data. This data was produced randomly in the `backend` service. The fact that you're seeing it in a web UI in the `frontend` means that the services are able to communicate. Unfortunately, this doesn't work out of the box on Linux
    right now due to how self-signed certificates are handled, please see the workaround [below](#troubleshooting)

### Next Steps

Now that you are able to run a multi-project application with [`tye run`](https://github.com/dotnet/tye/blob/main/docs/reference/commandline/tye-run.md), move on to [the next Exercise 2 (deploy)](#exercise-2-deploying-applications-to-a-kubernetes-cluster-quickly-and-easily) to learn how to deploy this application to Kubernetes.


### Troubleshooting

#### Certificate is invalid exception on Linux
`dotnet dev-certs ...` doesn't fully work on Linux so you need to generate and trust your own certificate.

##### Generate the certificate
```sh
# See https://stackoverflow.com/questions/55485511/how-to-run-dotnet-dev-certs-https-trust
# for more details

cat << EOF > localhost.conf
[req]
default_bits       = 2048
default_keyfile    = localhost.key
distinguished_name = req_distinguished_name
req_extensions     = req_ext
x509_extensions    = v3_ca

[req_distinguished_name]
commonName                  = Common Name (e.g. server FQDN or YOUR name)
commonName_default          = localhost
commonName_max              = 64

[req_ext]
subjectAltName = @alt_names

[v3_ca]
subjectAltName = @alt_names
basicConstraints = critical, CA:false
keyUsage = keyCertSign, cRLSign, digitalSignature,keyEncipherment

[alt_names]
DNS.1   = localhost
DNS.2   = 127.0.0.1

EOF

# Generate certificate from config
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout localhost.key -out localhost.crt \
    -config localhost.conf

# Export pfx
openssl pkcs12 -export -out localhost.pfx -inkey localhost.key -in localhost.crt

# Import CA as trusted
sudo cp localhost.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates 

# Validate the certificate
openssl verify localhost.crt
```

Once you have this working, copy `localhost.pfx` into the `backend` directory, then add the following
to `appsettings.json`

```json
{
  ...
  "Kestrel": {
    "Certificates": {
      "Default": {
        "Path": "localhost.pfx",
        "Password": ""
      }
    }
  }
}
```

You may still get an untrusted warning with your browser but it will work with dotnet.

# Exercise 2: Deploying applications to a Kubernetes cluster quickly and easily

## Getting Started with Deployment

This tutorial assumes that you have completed the [first step (run locally)](00_run_locally.md)

> :bulb: `tye` will use your current credentials for pushing Docker images and accessing kubernetes clusters. If you have configured kubectl with a context already, that's what [`tye deploy`](https://github.com/dotnet/tye/blob/main/docs/reference/commandline/tye-deploy.md) is going to use!

Before we deploy, make sure you have the following ready...

1. Installing [docker](https://docs.docker.com/install/) based on your operating system.

2. A container registry. Docker by default will create a container registry on [DockerHub](https://hub.docker.com/). You could also use [Azure Container Registry](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr) or another container registry of your choice, like a [local registry](https://docs.docker.com/registry/deploying/#run-a-local-registry) for testing.

    **Note: Pushing images to DockerHub is recommended for this tutorial**

3. A Kubernetes Cluster. There are many different options here, including:
    - [Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster)
    - [Kubernetes in Docker Desktop](https://www.docker.com/blog/docker-windows-desktop-now-kubernetes/), however it does take up quite a bit of memory on your machine, so use with caution.
    - [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
    - [K3s](https://k3s.io), a lightweight single-binary certified Kubernetes distribution from Rancher.
    - Another Kubernetes provider of your choice.

    **Note: For simplicity purposes, let's use Kubernetes in Docker Desktop**

> :warning: If you choose a container registry provided by a cloud provider (other than Dockerhub), you will likely have to take some steps to configure your kubernetes cluster to allow access. Follow the instructions provided by your cloud provider.

### Deploying the application

Now that we have our application running locally, let's deploy the application. In this example, we will deploy to Kubernetes by using `tye deploy`.

1. Deploy to Kubernetes

    Deploy the application by running:

    ```text
    tye deploy --interactive
    ```

    **Note: Do not specify a Kubernetes namespace since Tye has some bugs that do not allow to undeploy afterwards**
    > Enter the Container Registry (ex: 'example.azurecr.io' for Azure or 'example' for dockerhub):

    You will be prompted to enter your container registry. This is needed to tag images, and to push them to a location accessible by kubernetes.

    > :bulb: Under the hood `tye` uses `kubectl` and to execute deployments. In cases if you don't have `kubectl` installed or it's current context is invalid `tye deploy` will fail with the following error: "Drats! 'deploy' failed: Cannot apply manifests because kubectl is not installed." 

    If you are using dockerhub, the registry name will your dockerhub username. If you are a standalone container registry (for instance from your cloud provider), the registry name will look like a hostname, eg: `example.azurecr.io`.

    `tye deploy` does many different things to deploy an application to Kubernetes. It will:
    - Create a docker image for each project in your application.
    - Push each docker image to your container registry.
    - Generate a Kubernetes `Deployment` and `Service` for each project.
    - Apply the generated `Deployment` and `Service` to your current Kubernetes context.

2. Test it out!

    You should now see two pods running after deploying.

    ```text
    kubectl get pods
    ```

    ```text
    NAME                                             READY   STATUS    RESTARTS   AGE
    backend-ccfcd756f-xk2q9                          1/1     Running   0          85m
    frontend-84bbdf4f7d-6r5zp                        1/1     Running   0          85m
    ```

    You'll have two services in addition to the built-in `kubernetes` service.

    ```text
    kubectl get service
    ```

    ```text
    NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
    backend      ClusterIP   10.0.147.87   <none>        80/TCP    11s
    frontend     ClusterIP   10.0.20.168   <none>        80/TCP    14s
    kubernetes   ClusterIP   10.0.0.1      <none>        443/TCP   3d5h
    ```

    You can visit the frontend application by port forwarding to the frontend service.

    ```text
    kubectl port-forward svc/frontend 5000:80
    ```

    Now navigate to <http://localhost:5000> to view the frontend application working on Kubernetes. You should see the list of weather forecasts just like when you were running locally.

    > :bulb: Currently `tye` does not provide a way to expose pods/services created to the public internet. We'll add features related to `Ingress` in future releases.

    > :warning: Currently `tye` does not automatically enable TLS within the cluster, and so communication takes place over HTTP instead of HTTPS. This is typical way to deploy services in kubernetes - we may look to enable TLS as an option or by default in the future.

### Exploring tye.yaml

Tye has a optional configuration file (`tye.yaml`) to allow customizing settings. If you want to use `tye deploy` as part of a CI/CD system, it's expected that you'll have a `tye.yaml`.

1. Scaffolding `tye.yaml`

    Run the `tye init` command in the `microservices` directory to generate a default `tye.yaml`

    ```text
    tye init
    ```

    The contents of `tye.yaml` should look like:

    ```yaml
    # tye application configuration file
    # read all about it at https://github.com/dotnet/tye
    #
    # when you've given us a try, we'd love to know what you think:
    #    <survey link>
    #
    name: microservice
    services:
    - name: frontend
      project: frontend/frontend.csproj
    - name: backend
      project: backend/backend.csproj
    ```

    The top level scope (like the `name` node) is where global settings are applied.

    `tye.yaml` lists all of the application's services under the `services` node. This is the place for per-service configuration.

    See [schema](https://github.com/dotnet/tye/blob/main/docs/reference/schema.md) for more details about `tye.yaml`.

2. Adding a container registry to `tye.yaml`

    Based on what container registry you configured, add the following line in the `tye.yaml` file:

    ```yaml
    registry: <registry_name>
    ```

    If you are using dockerhub, the registry_name will your dockerhub username. If you are a standalone container registry (for instance from your cloud provider), the registry_name will look like a hostname, eg: `example.azurecr.io`.

    Now it's possible to use `tye deploy` without `--interactive` since the registry is stored as part of configuration.

    > :question: This step may not make much sense if you're using `tye.yaml` to store a personal Dockerhub username. A more typical use case would storing the name of a private registry for use in a CI/CD system.

### Undeploying the application

After deploying and playing around with the application, you may want to remove all resources associated from the Kubernetes cluster. You can remove resources by running:

```text
tye undeploy
```

This will remove all deployed resources. If you'd like to see what resources would be deleted, you can run:

```text
tye undeploy --what-if
```

### Next Steps

Now that you are able to deploy an application to Kubernetes, learn how to add a non-project dependency to tye.

# Exercise 3: Adding external services needed by the system and deploying them to a Kubernetes cluster

## Adding redis to an application

This tutorial assumes that you have completed the [first step (run locally)](00_run_locally.md) and [second step (deploy)](01_deploy.md).

We just showed how `tye` makes it easier to communicate between 2 applications running locally but what happens if we want to use redis to store weather information?

`Tye` can use `docker` to run images that run as part of your application. If you haven't already, make sure docker is installed on your operating system ([install docker](https://docs.docker.com/install/)) .


1. Change the `WeatherForecastController.Get()` method in the `backend` project to cache the weather information in redis using an `IDistributedCache`.

   Add the following `using`'s to the top of the file:

   ```C#
   using Microsoft.Extensions.Caching.Distributed;
   using System.Text.Json;
   ```

   And update `Get()`:

   ```C#
   [HttpGet]
   public async Task<string> Get([FromServices]IDistributedCache cache)
   {
       var weather = await cache.GetStringAsync("weather");

       if (weather == null)
       {
           var rng = new Random();
           var forecasts = Enumerable.Range(1, 5).Select(index => new WeatherForecast
           {
               Date = DateTime.Now.AddDays(index),
               TemperatureC = rng.Next(-20, 55),
               Summary = Summaries[rng.Next(Summaries.Length)]
           })
           .ToArray();

           weather = JsonSerializer.Serialize(forecasts);

           await cache.SetStringAsync("weather", weather, new DistributedCacheEntryOptions
           {
               AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(5)
           });
       }
       return weather;
   }
   ```

   This will store the weather data in Redis with an expiration time of 5 seconds.


2. Add a package reference to `Microsoft.Extensions.Caching.StackExchangeRedis` in the backend project:

   ```
   cd backend/
   dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
   cd ..
   ```

3. Modify `Startup.ConfigureServices` in the `backend` project to add the redis `IDistributedCache` implementation.
   ```C#
   public void ConfigureServices(IServiceCollection services)
   {
       services.AddControllers();

       services.AddStackExchangeRedisCache(o =>
       {
            o.Configuration = Configuration.GetConnectionString("redis");
        });
   }
   ```
   The above configures redis to the configuration string for the `redis` service injected by the `tye` host.

4. Modify `tye.yaml` to include redis as a dependency.

   > :bulb: You should have already created `tye.yaml` in a previous step near the end of the deployment tutorial.

   ```yaml
   name: microservice
   registry: <registry_name>
   services:
   - name: backend
     project: backend\backend.csproj
   - name: frontend
     project: frontend\frontend.csproj
   - name: redis
     image: redis
     bindings:
     - port: 6379
       connectionString: "${host}:${port}" 
   - name: redis-cli
     image: redis
     args: "redis-cli -h redis MONITOR"
   ```

    We've added 2 services to the `tye.yaml` file. The `redis` service itself and a `redis-cli` service that we will use to watch the data being sent to and retrieved from redis.

    > :bulb: The `"${host}:${port}"` format in the `connectionString` property will substitute the values of the host and port number to produce a connection string that can be used with StackExchange.Redis.

5. Run the `tye` command line in the solution root

   > :bulb: Make sure your command-line is in the `microservice/` directory. One of the previous steps had you change directories to edit a specific project.

   ```
   tye run
   ```

   Navigate to <http://localhost:8000> to see the dashboard running. Now you will see both `redis` and the `redis-cli` running listed in the dashboard.
   
   Navigate to the `frontend` application and verify that the data returned is the same after refreshing the page multiple times. New content will be loaded every 5 seconds, so if you wait that long and refresh again, you should see new data. You can also look at the `redis-cli` logs using the dashboard and see what data is being cached in redis.

### Deploying redis
   
1. Deploy redis to Kubernetes

    `tye deploy` will not deploy the redis configuration, so you need to deploy it first. Run:

    ```text
    kubectl apply -f https://raw.githubusercontent.com/dotnet/tye/main/docs/tutorials/hello-tye/redis.yaml
    ```

    or use `K8S Lens` to create the resources (use the file `redis.yaml`, create a new resource using K8S Lens and paste the content of the file).
    
    This will create a deployment and service for redis. You can see that by running:

    ```text
    kubectl get deployments
    ```

    You will see redis deployed and running.

2. Deploy to Kubernetes

    Next, deploy the rest of the application by running:

    ```text
    tye deploy --interactive
    ```

    You'll be prompted for the connection string for redis. 

    ```text
    Validating Secrets...
        Enter the connection string to use for service 'redis':
    ```

    Enter the following to use instance that you just deployed:

    ```text
    redis:6379
    ```

    `tye deploy` will create kubernetes secret to store the connection string.

    ```text
    Validating Secrets...
        Enter the connection string to use for service 'redis': redis:6379
        Created secret 'binding-production-redis-secret'.
    ```

    > :question: `--interactive` is needed here to create the secret. This is a one-time configuration step. In a CI/CD scenario you would not want to have to specify connection strings over and over, deployment would rely on the existing configuration in the cluster.

    > :bulb: Tye uses kubernetes secrets to store connection information about dependencies like redis that might live outside the cluster. Tye will automatically generate mappings between service names, binding names, and secret names.

3. Test it out!

    You should now see three pods running after deploying.

    ```text
    kubectl get pods
    ```

    ```
    NAME                                             READY   STATUS    RESTARTS   AGE
    backend-ccfcd756f-xk2q9                          1/1     Running   0          85m
    frontend-84bbdf4f7d-6r5zp                        1/1     Running   0          85m
    redis-5f554bd8bd-rv26p                           1/1     Running   0          98m
    ```

    Just like last time, we'll need to port-forward to access the `frontend` from outside the cluster.

    ```text
    kubectl port-forward svc/frontend 5000:80
    ``` 

    Visit `http://localhost:5000` to see the `frontend` working in kubernetes.

4. Clean-up

    At this point, you may want to undeploy the application by running `tye undeploy`.
