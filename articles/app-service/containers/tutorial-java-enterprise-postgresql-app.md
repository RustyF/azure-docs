---
title: Build a Java Enterprise web app in Azure App Service on Linux | Microsoft Docs 
description: Learn how to get a Java Enterprise app working in Wildfly on Azure App Service on Linux.
author: JasonFreeberg
manager: routlaw

ms.service: app-service-web
ms.workload: web
ms.tgt_pltfrm: na
ms.devlang: java
ms.topic: tutorial
ms.date: 11/13/2018
ms.author: jafreebe
---

# Tutorial: Build a Java EE and Postgres web app in Azure

This tutorial will show you how to create a Java Enterprise Edition (EE) web app on Azure App Service and connect it to a Postgres database. When you are finished, you will have a [WildFly](http://www.wildfly.org/about/) application storing data in [Azure Database for Postgres](https://azure.microsoft.com/services/postgresql/) running on Azure [App Service for Linux](app-service-linux-intro.md).

In this tutorial, you will learn how to:
> [!div class="checklist"]
> * Deploy a Java EE app to Azure using Maven
> * Create a Postgres database in Azure
> * Configure the WildFly server to use Postgres
> * Update and redeploy the app
> * Run unit tests on WildFly

## Prerequisites

1. [Download and install Git](https://git-scm.com/)
1. [Download and install Maven 3](https://maven.apache.org/install.html)
1. [Download and install the Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)

## Clone and edit the sample app

In this step, you will clone the sample application and configure the Maven Project Object Model (POM or pom.xml) for deployment.

### Clone the sample

In the terminal window, navigate to a working directory and clone [the sample repository](https://github.com/Azure-Samples/wildfly-petstore-quickstart).

```bash
git clone https://github.com/Azure-Samples/wildfly-petstore-quickstart.git
```

### Update the Maven POM

Update the Maven POM with the desired name and resource group of your App Service. These values will be injected into the Azure plugin, which is further down in _pom.xml_ file. You do not need to create the App Service plan or instance beforehand. The Maven plugin will create the resource group and App Service if it does not already exist.

Replace the placeholders with your desired resource names:
```xml
<azure.plugin.appname>YOUR_APP_NAME</azure.plugin.appname>
<azure.plugin.resourcegroup>YOUR_RESOURCE_GROUP</azure.plugin.resourcegroup>
```

You can scroll down to the `<plugins>` section of _pom.xml_ to inspect the Azure plugin.

## Build and deploy the application

We will now use Maven to build our application and deploy it to App Service.

### Build the .war file

The POM in this project is configured to package the application into a Web Archive (WAR) file. Build the application using Maven:

```bash
mvn clean install -DskipTests
```

The test cases in this application are designed to be run when the application is deployed onto WildFly. We will skip the tests to build locally, and you run the tests once the application is deployed onto App Service.

### Deploy to App Service

Now that the WAR is ready, we can use the Azure plugin to deploy to App Service:

```bash
mvn azure-webapp:deploy
```

When the deployment finishes, continue to the next step.

### Create a record

Open a browser and navigate to `https://<your_app_name>.azurewebsites.net/`. Congratulations, you have deployed a Java EE application to Azure App Service!

At this point, the application is using an in-memory H2 database. Click "admin" in the navigation bar and create a new category. The record in your in-memory database will be lost if you restart your App Service instance. In the following steps, you will fix this by provisioning a Postgres database on Azure and configure WildFly to use it.

![Admin button location](media/tutorial-java-enterprise-postgresql-app/admin_button.JPG)

## Provision a Postgres Database

To provision a Postgres database server, open a terminal and run the following command with your desired values for the server name, username, password, and location. Use the same resource group that your App Service is in. Keep a note of your password for later!

```bash
az postgres server create -n <desired-name> -g <same-resource-group> --sku-name GP_Gen4_2 -u <desired-username> -p <desired-password> -l <location>
```

Browse to the Portal and search for your Postgres database. When the blade is up, copy the "Server name" and "Server admin login name" values, you will need them later.

### Allow access to Azure Services

In the **Connection security** panel of the Azure Database blade, toggle the "Allow access to Azure services" button to the **ON** position.

![Allowing access to Azure Services](media/tutorial-java-enterprise-postgresql-app/postgress_button.JPG)

## Update your Java app for Postgres

We will now make some changes to the Java application to enable it to use our Postgres database.

### Add Postgres credentials to the POM

In _pom.xml_ replace the placeholder values with your Postgres server name, admin login name, and password. These values will be injected as environment variables in your App Service instance when you redeploy the application.

```xml
<azure.plugin.postgres-server-name>SERVER_NAME</azure.plugin.postgres-server-name>
<azure.plugin.postgres-username>USERNAME@FIRST_PART_OF_SERVER_NAME</azure.plugin.postgres-username>
<azure.plugin.postgres-password>PASSWORD</azure.plugin.postgres-password>
```

### Update the Java Transaction API

Next, we need to edit our Java Transaction API (JPA) configuration so that our Java application will communicate with Postgres instead of the in-memory H2 database we were using previously. Open an editor to _src/main/resources/META-INF/persistence.xml_. Replace the value for `<jta-data-source>` with `java:jboss/datasources/postgresDS`. Your JTA XML should now have this setting:

```xml
...
<jta-data-source>java:jboss/datasources/postgresDS</jta-data-source>
...
```

## Configure the WildFly application server

Before deploying our reconfigured application, we must update the WildFly application server with the Postgres module and its dependencies. To configure the server, we will need the four files in the  `wildfly_config/` directory:

- **postgresql-42.2.5.jar**: This JAR file is the JDBC driver for Postgres. For more information,  see the [official website](https://jdbc.postgresql.org/index.html).
- **postgres-module.xml**: This XML file declares a name for the Postgres module (org.postgres). It also specifies the resources and dependencies necessary for the module to be used.
- **jboss_cli_commands.cl**: This file contains configuration commands that will be executed to by the JBoss CLI. The commands add the Postgres module to the WildFly application server, provide the credentials, declare a JNDI name, set the timeout threshold, etc. If you are unfamiliar with the JBoss CLI, see the [official documentation](https://access.redhat.com/documentation/red_hat_jboss_enterprise_application_platform/7.0/html-single/management_cli_guide/#how_to_cli).
- **startup_script.sh**: Finally, this shell script will be executed whenever your App Service instance is started. The script only performs one function: piping the commands in `jboss_cli_commands.cli` to the JBoss CLI.

We highly suggest reading the contents of these files, especially _jboss_cli_commands.cli_.

### FTP the configuration files

We will need to FTP the contents of `wildfly_config/` to our App Service instance. To get your FTP credentials, click the **Get Publish Profile** button on the App Service blade in the Azure portal. Your FTP username and password will be in the downloaded XML document. For more information on the Publish Profile,  see [this document](https://docs.microsoft.com/azure/app-service/app-service-deployment-credentials).

Using an FTP tool of your choice, transfer the four files in `wildfly_config/` to `/home/site/deployments/tools/`. (Note that you should not transfer the directory, just the files themselves.)


### Finalize App Service

In the App Service blade navigate to the "Application settings" panel. Under "Runtime", set the "Startup File" field to `/home/site/deployments/tools/startup_script.sh`. This will ensure that the shell script is run after the App Service instance is created, but before the WildFly server starts.

Finally, restart your App Service. The button is in the "Overview" panel.

## Redeploy the application

In a terminal window, rebuild and redeploy your application.

```bash
mvn clean install -DskipTests azure-webapp:deploy
```

Congratulations! Your application is now using a Postgres database and any records created in the application will be stored in Postgres, rather than the previous H3 in-memory database. To confirm this, you can make a record and restart your App Service. The records will still be there when your application restarts.

## Clean up

If you don't need these resources for another tutorial (see Next steps), you can delete them by running the following command:

```bash
az group delete --name <your_resource_group> 
```

## Next steps

Now that you have a Java EE application deployed to App Service, please see the [Java Enterprise developer guide](https://aka.ms/wildfly-quickstart) for more information on setting up services, troubleshooting, and scaling your application.