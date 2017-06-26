# AppDev workshop

- Developing with Spring Cloud          [Lecture]

- Service Registration and Discovery    [Lecture]

- Simple Discoverable applications      [Lab]

- Service Discovery in the Cloud        [Lab]

- Zero-Downtime Deployments for Discoverable services [Lab]

- Configuration Management              [Lecture]

- Configuration Management in the Cloud [Lab]

- Zuul                                  [Lecture / Lab]


<p>
<p>

## Service Registration and Discovery    [Lecture]

<a href="docs/SpringCloudServiceDiscovery.pdf">Slides</a>

- SCS gives the ability to have our applications talk each directly  without going thru the router. To do that we need have an application setting called  `spring.cloud.services.registrationMethod`. The values for this setting are : `route` and `direct`. If we use `route` (the default value), our applications register using their PCF route else if they register using their IP address.

```
NOTE: To enable direct registration, you must configure the PCF environment to allow traffic across containers or cells. In PCF 1.6, visit the Pivotal Cloud Foundry® Operations Manager®, click the Pivotal Elastic Runtime tile, and in the Security Config tab, ensure that the “Enable cross-container traffic” option is enabled.
```

##  Service Registration and Discovery    [Lab]

```
cf-demo-client ----{ http://demo/hi?name=Bob }--> cf-demo-app
                <----{ `hello Bob` }-------------
```

First we are going to get our 2 applications running ~~locally~~. With our local Eureka server.
And the second part of the lab is to push our 2 applications to PCF and use PCF Service Registry to register our applications rather than our standalone Eureka server.

### Set up

1. You will need JDK 8, Maven and git. We will use the command line to launch the apps (`mvn spring-boot:run or java -jar`).
2. git clone https://github.com/nagelpat/spring-cloud-workshop/

Go to the folder, `labs/lab1` in the cloned git repo.

### Service Discovery in the Cloud

**Create Eureka or Service Registry in PCF**

1. create service (http://docs.pivotal.io/spring-cloud-services/service-registry/creating-an-instance.html)

  `cf marketplace -s p-service-registry`

  `cf create-service p-service-registry standard registry-service`

  `cf services`

  Make sure the service is ready. PCF provisions the services asynchronously.

2. Go to the AppsManager and check the service. Check out the dashboard.

**Deploy cf-demo-app in PCF**

1. check the manifest.yml (e.g. rename the host : `yourname-cf-app-demo`, or `cf-app-demo-${random-word}`. )
2. push the dmeo application (check out url, domain) with the manifest.yml

  `cf push`

3. Check the app is working

  `curl https://URI/hello?name=Marcial`

4. Check the credentials PCF has injected into our application

  `cf env cf-demo-app`

  You will get something like this because we have stated in the `manifest.yml` that our application needs a service called `registry-service`. PCF takes care of injecting the credentials from the `registry-service` into the application's environment.

  ```
  {
   "VCAP_SERVICES": {
    "p-service-registry": [
     {
      "credentials": {
       "access_token_uri": "https://p-spring-cloud-services.uaa.system-dev.chdc20-cf.pez.com/oauth/token",
       "client_id": "p-service-registry-XXXXXXXX",
       "client_secret": "XXXXXXX",
       "uri": "https://eureka-044ac701-7919-4373-ad76-bdd0743fd813.apps-dev.chdc20-cf.pez.com"
      },
      "label": "p-service-registry",
      "name": "registry-service",
      "plan": "standard",
      "provider": null,
      "syslog_drain_url": null,
      "tags": [
       "eureka",
       "discovery",
       "registry",
       "spring-cloud"
      ],
      "volume_mounts": []
     }
    ]
   }
  }
  ```
  Thanks to a Spring project called [Spring Cloud Connectors ](http://cloud.spring.io/spring-cloud-connectors/) and [Spring Cloud Connectors for Cloud Foundry](http://cloud.spring.io/spring-cloud-connectors/spring-cloud-cloud-foundry-connector.html), Spring Cloud Eureka is configured from the VCAP_SERVICES. If you want to know how you can start looking at [here](https://github.com/pivotal-cf/spring-cloud-services-connector/blob/master/spring-cloud-services-cloudfoundry-connector/src/main/java/io/pivotal/spring/cloud/cloudfoundry/EurekaServiceInfoCreator.java)

4. Go to the Dashboard  of the registry-service and check that our service is there
5. Scale the app to 2 instances. Check the dashboard.

**Deploy cf-demo-client in PCF**

1. install our client application
2. check the manifest.yml
3. push the application with the manifest
4. Check the app is working
  `curl 'https://URI/hi?name=Marcial'`
  
5. Check that our app is not actually registered with Eureka however it has discovered our `demo` app. Also, see the instanceId shown in the Registry's dashboard.

6. We can rely on RestTemplate to automatically resolve a service-name to a url. But we can also use the Discovery API to get their urls.
`curl https://URI-demo-client/service-instances/demo | jq .`

## Configuration Management [Lecture]

[Slides](docs/SpringCloudConfigSlides.pdf)

Very interesting talk  [Implementing Config Server and Extending It](https://www.infoq.com/presentations/config-server-security)

### Additional comments
- We can store our credentials encrypted in the repo and Spring Config Server will decrypt them before delivering them to the client.
- Spring Config Service (PCF Tile) does not support server-side decryption. Instead, we have to configure our client to do it. For that we need to make sure that the java buildpack is configured with `Java Cryptography Extension (JCE) Unlimited Strength policy files`. For further details check out the <a href="http://docs.pivotal.io/spring-cloud-services/config-server/writing-client-applications.html#use-client-side-decryption">docs</a>.
- We should really use <a href="https://spring.io/blog/2016/06/24/managing-secrets-with-vault">Spring Cloud Vault</a> integrated with Spring Config Server to retrieve secrets.

## Configuration Management  [Lab]
Go to the folder, labs/lab2 in the cloned git repo.

1. Check the config server in the marketplace
`cf marketplace -s p-config-server`
2. Create a service instance
`cf create-service -c '{"git": { "uri": "https://github.com/nagelpat/spring-cloud-workshop-config" }, "count": 1 }' p-config-server standard config-server`

3. Make sure the service is available (`cf services`)
3. Modify our application so that it has a `bootstrap.yml` rather than `application.yml`. We don't really need an `application.yml`. If we have one, Spring Config client will take that as the default properties of the application.

4. Our repo has already our `demo.yml`. If we did not have our the setting `spring.application.name`, the `spring-auto-configuration` jar injected by the java buildpack will automatically create a `spring.application.name` environment variable based on the env variable `VCAP_APPLICATION { ... "application_name": "cf-demo-app" ... }`.

5. Push our `cf-demo-app`.

6. Check that our application is now bound to the config server
`cf env cf-demo-app`

7. Check that it loaded the application's configuration from the config server.
`curl cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/env | jq .`

We should have these configuration at the top :
```
"configService:https://github.com/MarcialRosales/spring-cloud-workshop-config/demo.yml": {
    "mymessage": "Good afternoon"
  },
  "configService:https://github.com/MarcialRosales/spring-cloud-workshop-config/application.yml": {
    "info.id": "${spring.application.name}"
  },
```  

8. Check that our application is actually loading the message from the central config and not the default message `Hello`.
`curl cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/hello?name=Marcial`

9. We can modify the demo.yml in github, and ask our application to reload the settings.
`curl -X POST cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/refresh`

Check the message again.
`curl cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/hello?name=Marcial`


10. Add a new configuration for production : `demo-production.yml` to the repo.

11. Configure our application to use production profile by manually setting an environment variable in CF:
`cf set-env cf-demo-app SPRING_PROFILES_ACTIVE production`

we have to restage our application because we have modified the environment.

12. Check our application returns us a different value this type
`curl cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/env | jq .`

We should have these configuration at the top :


## Configuration Management - Dynamically reconfigure your application [Lab]

### Dynamically changing settings at runtime in the application
- Spring will automatically rebuild a `@Bean` which is either annotated as `@RefreshScope` or `@ConfigurationProperties`. Spring assumes that a `ConfigurationProperties` is something we most likely want to refresh if any of its settings change.
- Spring will rebuild a bean when it receives a signal such as `RefreshRemoteApplicationEvent`. Spring config server sends this event to the application when it receives an event from the remote repo (via webhooks) or when it detects a change in the local filesystem -assuming we are using `native` repo.

Note about Reloading configuration: This works provided you only have one instance. Ideally, we want to configure our config server to receive a callback from Github (webhooks onto the actuator endpoint `/monitor`) when a change occurs. The config server (if bundled with the jar `spring-cloud-config-monitor`).
If we have more than one application instances you can still reload the configuration on all instances if you add the dependency `spring-cloud-starter-bus-amqp` to all the applications. It exposes a new endpoint called `/bus/refresh` . If we call that endpoint in one of the application instances, it propagates a refresh request to other instances.

Logging level is the most common settings we want to adjust at runtime. An Exercise is to modify the code to add a logger and add the logging level the demo.yml or demo-production.yml :
```
logging:
  level:
    io.pivotal.demo.CfDemoAppApplication: debug    

```


## Resiliency - What do we do if Config server is down?
We can either fail fast which means our applications fail to start. PCF would try to deploy the application a few times before giving up.
`spring.cloud.config.failFast=true`. Or if we can retry a few times before failing to start: `spring.cloud.config.failFast=false`, add the following dependencies to your project `spring-retry` and `spring-boot-starter-aop` and configure the retry mechanism via  `spring.cloud.config.retry.*`.

What happens if Config server cannot connect to the remote repo? @TODO test it!


## How to organize my application's configuration around the concept of a central repository [Lab]

#### Get started very quickly with spring config server : local file system (no git repo required)
```
---
spring.profiles: native
spring:
  cloud:
    config:
      server:
        native:
          searchLocations: ../../spring-cloud-workshop-config      
```

#### Use local git repo (all files must be committed!). One repo for all our applications and each application and profile has its own folder.
```
---
spring.profiles: git-local-common-repo
spring:
  cloud:
    config:
      server:
        git:
          uri: file:../../spring-cloud-workshop-config
          searchPaths: groupA-{application}-{profile}
```

#### Use local git repo. But different repos for different profiles
Spring Config server will try to resolve a pattern against ${application}/{profile}

```
---
spring.profiles: git-local-multi-repos-per-profile
spring:
  cloud:
    config:
      server:
        git:
          uri: file:../../emptyRepo
          repos:
            dev-repos:
              pattern: "*/dev"
              uri: file:../../dev-repo
            prod-repos:
              pattern: "*/prod"
              uri: file:../../prod-repo
```
 In this case, we have decided to have one repo specific for dev profile and another for prod profile              
 `curl localhost:8888/quote-service2/dev | jq .`

#### Use local git repo. Multiple repos per teams.

```
 ---
 spring.profiles: git-local-multi-repos-per-teams
 spring:
   cloud:
     config:
       server:
         git:
           uri: file:../../emptyRepo
           repos:
             trading:
               pattern: trading-*
               uri: file:../../trading
             pricing:
               pattern: pricing-*
               uri: file:../../pricing
             orders:
               pattern: orders-*
               uri: file:../../orders

```

We have 3 teams, trading, pricing, and orders. One repo per team responsible of a business capability.               
`curl localhost:8888/trading-execution-service/default | jq .`
`curl localhost:8888/pricing-quote-service/default | jq .`

#### Use local git repo. One repo per application.
```
---
spring.profiles: git-local-one-repo-per-app
spring:
  cloud:
    config:
      server:
        git:
          uri: file:../../{application}-repo
```
