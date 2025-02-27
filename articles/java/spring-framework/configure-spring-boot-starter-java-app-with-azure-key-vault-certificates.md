---
title: "Tutorial: Secure Spring Boot apps using Azure Key Vault certificates"
description: In this tutorial, you secure your Spring Boot (including Azure Spring Apps) apps with TLS/SSL certificates using Azure Key Vault and managed identities for Azure resources.
ms.date: 03/30/2022
ms.service: key-vault
ms.topic: tutorial
ms.custom: devx-track-java, devx-track-azurecli
ms.author: edburns
---

# Tutorial: Secure Spring Boot apps using Azure Key Vault certificates

This tutorial shows you how to secure your Spring Boot (including Azure Spring Apps) apps with TLS/SSL certificates using Azure Key Vault and managed identities for Azure resources.

Production-grade Spring Boot applications, whether in the cloud or on-premises, require end-to-end encryption for network traffic using standard TLS protocols. Most TLS/SSL certificates you come across are discoverable from a public root certificate authority (CA). Sometimes, however, this discovery isn't possible. When certificates aren't discoverable, the app must have some way to load such certificates, present them to inbound network connections, and accept them from outbound network connections.

Spring Boot apps typically enable TLS by installing the certificates. The certificates are installed into the local key store of the JVM that's running the Spring Boot app. With Spring on Azure, certificates are not installed locally. Instead, Spring integration for Microsoft Azure provides a secure and frictionless way to enable TLS with help from Azure Key Vault and managed identity for Azure resources.

:::image type="content" source="media/configure-spring-boot-starter-java-app-with-azure-key-vault-certificates/spring-to-azure-key-vault-certificates.svg" alt-text="Diagram showing interaction of elements in this tutorial." border="false":::

In this tutorial, you learn how to:

> [!div class="checklist"]
> * Create a GNU/Linux VM with system-assigned managed identity
> * Create an Azure Key Vault
> * Create a self-signed TLS/SSL certificate
> * Store the self-signed TLS/SSL certificate in the Azure Key Vault
> * Run a Spring Boot application where the TLS/SSL certificate for inbound connections comes from Azure Key Vault
> * Run a Spring Boot application where the TLS/SSL certificate for outbound connections comes from Azure Key Vault

## Prerequisites

- [!INCLUDE [free subscription](../../includes/quickstarts-free-trial-note.md)]
[!INCLUDE [curl](includes/prerequisites-curl.md)]
[!INCLUDE [jq](includes/prerequisites-jq.md)]
[!INCLUDE [Azure CLI](includes/prerequisites-azure-cli.md)]

- A supported Java Development Kit (JDK), version 8. For more information, see [Java support on Azure and Azure Stack](../fundamentals/java-support-on-azure.md).

- [Apache Maven](http://maven.apache.org/), version 3.0.

> [!IMPORTANT]
> Spring Boot version 2.5 or 2.6 is required to complete the steps in this article.

## Create a GNU/Linux VM with system-assigned managed identity

Use the following steps to create an Azure VM with a system-assigned managed identity and prepare it to run the Spring Boot application. For an overview of managed identities for Azure resources, see [What are managed identities for Azure resources?](/azure/active-directory/managed-identities-azure-resources/overview).

1. Open a Bash shell.

1. Sign out and delete some authentication files to remove any lingering credentials.

   ```azurecli
   az logout
   rm ~/.azure/accessTokens.json
   rm ~/.azure/azureProfile.json
   ```

1. Sign in to your Azure CLI.

   ```azurecli
   az login
   ```

1. Set the subscription ID. Be sure to replace the placeholder with the appropriate value.

   ```azurecli
   az account set -s <your subscription ID>
   ```

1. Create an Azure resource group. Take note of the resource group name for later use.

   ```azurecli
   az group create \
       --name <your resource group name> \
       --location <your resource group region>
   ```

1. Set default resource group.

   ```azurecli
   az configure --defaults group=<your resource group name>
   ```

1. Create the VM instance with system-assigned managed identity enabled, using the image `UbuntuLTS` provided by `UbuntuServer`.

   ```azurecli
   az vm create \
       --name <your VM name> \
       --debug \
       --generate-ssh-keys \
       --assign-identity \
       --image UbuntuLTS \
       --admin-username azureuser
   ```

   In the JSON output, note down the value of the `publicIpAddress` and `systemAssignedIdentity` properties. You'll use these values later in the tutorial.

   > [!NOTE]
   > The name `UbuntuLTS` is a Uniform Resource Name (URN) alias, which is a shortened version created for popular images like *UbuntuLTS*.
   > Run the following command to display a cached list of popular images in table format:
   >
   > ```azurecli
   > az vm image list --output table
   > ```

1. Install the Microsoft OpenJDK. For complete information about OpenJDK, see [Microsoft Build of OpenJDK](/java/openjdk).

   ```shell
   ssh azureuser@<your VM public IP address>
   ```

   Add the repository. Replace the version placeholder in following commands and execute:

   ```shell
   wget https://packages.microsoft.com/config/ubuntu/{version}/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
   sudo dpkg -i packages-microsoft-prod.deb
   ```

   Install the Microsoft Build of OpenJDK by running the following commands:

   ```shell
   sudo apt install apt-transport-https
   sudo apt update
   sudo apt install msopenjdk-11
   ```

   > [!NOTE]
   > Another, faster way to get set up the certified Azul Zulu for Azure – Enterprise Edition VM image, which can avoid the installation of Azure SDK. For complete information about Azul Zulu for Azure, see [Download Java for Azure](https://www.azul.com/downloads/azure-only/zulu/).
   > Run the following command to obtain the URN name.
   >
   > ```azurecli
   >    az vm image list \
   >        --offer Zulu \
   >        --location <your region> \
   >        --all | grep urn
   > ```
   >
   > This command may take a while to complete. When the command completes, it produces output similar to the following lines. Select the value for JDK 11 on Ubuntu.
   >
   > ```output
   > "urn": "azul:azul-zulu11-ubuntu-2004:zulu-jdk11-ubtu2004:20.11.0",
   > ...
   > "urn": "azul:azul-zulu8-ubuntu-2004:zulu-jdk8-ubtu2004:20.11.0",
   > "urn": "azul:azul-zulu8-windows-2019:azul-zulu8-windows2019:20.11.0",
   > ```
   >
   > Use the following command to accept the terms for the image to allow the VM to be created.
   >
   > ```azurecli
   > az vm image terms accept --urn azul:azul-zulu11-ubuntu-2004:zulu-jdk11-ubtu2004:20.11.0
   > ```
>

## Create and configure an Azure Key Vault

Use the following steps to create an Azure Key Vault, and to grant permission for the VM's system-assigned managed identity to access the Key Vault for certificates.

1. Create an Azure Key Vault within the resource group.

   ```azurecli
   az keyvault create \
       --name <your Key Vault name> \
       --location <your resource group region>
   export KEY_VAULT_URI=$(az keyvault show --name <your Key Vault name> | jq -r '.properties.vaultUri')
   ```

   Take note of the `KEY_VAULT_URI` value. You'll use it later.

1. Grant the VM permission to use the Key Vault for certificates.

   ```azurecli
   az keyvault set-policy \
       --name <your Key Vault name> \
       --object-id <your system-assigned identity> \
       --secret-permissions get list \
       --certificate-permissions get list import
   ```

## Create and store a self-signed TLS/SSL certificate

The steps in this tutorial apply to any TLS/SSL certificate (including self-signed) stored directly in Azure Key Vault. Self-signed certificates aren't suitable for use in production, but are useful for dev and test applications. This tutorial uses a self-signed certificate. To create the certificate, use the following command.

```azurecli
az keyvault certificate create \
    --vault-name <your Key Vault name> \
    --name mycert \
    --policy "$(az keyvault certificate get-default-policy)"
```

## Run a Spring Boot application with secure inbound connections

In this section, you'll create a Spring Boot starter application where the TLS/SSL certificate for inbound connections comes from Azure Key Vault.

To create the application, use the following steps:

1. Browse to <https://start.spring.io/>.
1. Select the choices as shown in the picture following this list.
    * **Project**: **Maven Project**
    * **Language**: **Java**
    * **Spring Boot**: **2.5.10**
    * **Group**: *com.contoso* (You can put any valid Java package name here.)
    * **Artifact**: *ssltest* (You can put any valid Java class name here.)
    * **Packaging**: **Jar**
    * **Java**: **11**
1. Select **Add Dependencies...**.
1. In the text field, type *Spring Web* and press Ctrl+Enter.
1. In the text field type *Azure Support* and press Enter. Your screen should look like the following.

   :::image type="content" source="media/spring-initializer/2.5.11/mvn-java8-azure-web.png" alt-text="Screenshot of Spring Initializr with basic options.":::

1. At the bottom of the page, select **Generate**.
1. When prompted, download the project to a path on your local computer. This tutorial uses an *ssltest* directory in the current user's home directory. The values above will give you an *ssltest.zip* file in that directory.

### Enable the Spring Boot app to load the TLS/SSL certificate

To enable the app to load the certificate, use the following steps:

1. Unzip the *ssltest.zip* file.

1. Remove the *test* directory and its subdirectories. This tutorial ignores the test, so you can safely delete the directory.

1. Rename *application.properties* in *src/main/resources* to *application.yml*.

1. The file layout will look like the following.

   ```
   ├── HELP.md
   ├── mvnw
   ├── mvnw.cmd
   ├── pom.xml
   └── src
       └── main
           ├── java
           │   └── com
           │       └── contoso
           │           └── ssltest
           │               └── SsltestApplication.java
           └── resources
               ├── application.yml
               ├── static
               └── templates
   ```

1. Modify the POM to add a dependency on `azure-spring-boot-starter-keyvault-certificates`. Add the following code to the `<dependencies>` section of the *pom.xml* file.

   ```xml
   <dependency>
      <groupId>com.azure.spring</groupId>
      <artifactId>azure-spring-boot-starter-keyvault-certificates</artifactId>
   </dependency>
   ```

1. Edit the *src/main/resources/application.yml* file so that it has the following contents.

   ```yaml
   server:
     ssl:
       key-alias: <the name of the certificate in Azure Key Vault to use>
       key-store-type: AzureKeyVault
       trust-store-type: AzureKeyVault
     port: 8443
   azure:
     keyvault:
       uri: <the URI of the Azure Key Vault to use>
   ```

   These values enable the Spring Boot app to perform the *load* action for the TLS/SSL certificate, as mentioned at the beginning of the tutorial. The following table describes the property values.

   | Property Name | Explanation |
   |---------------|-------------|
   |server.port|The local TCP port on which to listen for HTTPS connections.|
   |server.ssl.key-alias|The value of the `--name` argument you passed to `az keyvault certificate create`.|
   |server.ssl.key-store-type|Must be `AzureKeyVault`.|
   |server.ssl.trust-store-type|Must be `AzureKeyVault`.|
   |azure.keyvault.uri|The `vaultUri` property in the return JSON from `az keyvault create`. You saved this value in an environment variable.|

   The only property specific to Key Vault is `azure.keyvault.uri`. The app is running on a VM whose system-assigned managed identity has been granted access to the Key Vault. Therefore, the app has also been granted access.

These changes enable the Spring Boot app to load the TLS/SSL certificate. In the next section, you'll enable the app to perform the *accept* action for the TLS/SSL certificate, as mentioned at the beginning of the tutorial.

### Create a Spring Boot REST controller

To create the REST controller, use the following steps:

1. Edit the *src/main/java/com/contoso/ssltest/SsltestApplication.java* file so that it has the following contents.

   ```java
   package com.contoso.ssltest;

   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;

   @SpringBootApplication
   @RestController
   public class SsltestApplication {

       public static void main(String[] args) {
           SpringApplication.run(SsltestApplication.class, args);
       }

       @GetMapping(value = "/ssl-test")
       public String inbound(){
           return "Inbound TLS is working!!";
       }

       @GetMapping(value = "/exit")
       public void exit() {
           System.exit(0);
       }

   }
   ```

   Calling `System.exit(0)` from within an unauthenticated REST GET call is only for demonstration purposes. Don't use `System.exit(0)` in a real application.

   This code illustrates the *present* action mentioned at the beginning of this tutorial. The following list highlights some details about this code:

    * There's now a `@RestController` annotation on the `SsltestApplication` class generated by Spring Initializr.
    * There's a method annotated with `@GetMapping`, with a `value` for the HTTP call you'll make.
    * The `inbound` method simply returns a greeting when a browser makes an HTTPS request to the `/ssl-test` path. The `inbound` method illustrates how the server presents the TLS/SSL certificate to the browser.
    * The `exit` method will cause the JVM to exit when invoked. This method is a convenience to make the sample easy to run in the context of this tutorial.

1. Open a new Bash shell and navigate to the *ssltest* directory. Run the following command.

   ```bash
   mvn clean package
   ```

   Maven compiles the code and packages it up into an executable JAR file

1. Verify that the network security group created within `<your resource group name>` allows inbound traffic on ports 22 and 8443 from your IP address. To learn about configuring network security group rules to allow inbound traffic, see the [Work with security rules](/azure/virtual-network/manage-network-security-group#work-with-security-rules) section of [Create, change, or delete a network security group](/azure/virtual-network/manage-network-security-group).

1. Put the executable JAR file on the VM.

   ```bash
   cd target
   sftp azureuser@<your VM public IP address>
   put *.jar
   ```

### Run the app on the server

Now that you've built the Spring Boot app and uploaded it to the VM, use the following steps to run it on the VM and call the REST endpoint with curl.

1. Use SSH to connect to the VM, then run the executable jar.

   ```bash
   set -o noglob
   ssh azureuser@<your VM public IP address> "java -jar *.jar"
   ```

1. Open a new Bash shell and execute the following command to verify that the server presents the TLS/SSL certificate.

   ```bash
   curl --insecure https://<your VM public IP address>:8443/ssl-test
   ```

1. Invoke the `exit` path to kill the server and close the network sockets.

   ```bash
   curl --insecure https://<your VM public IP address>:8443/exit
   ```

Now that you've seen the *load* and *present* actions with a self-signed TLS/SSL certificate, you'll make some trivial changes to the app to see the *accept* action as well.

## Run a Spring Boot application with secure outbound connections

In this section, you'll modify the code in the previous section so that the TLS/SSL certificate for outbound connections comes from Azure Key Vault. Therefore, the *load*, *present*, and *accept* actions are satisfied from the Azure Key Vault.

### Modify the SsltestApplication to illustrate outbound TLS connections

Use the following steps to modify the application:

1. Add the dependency on Apache HTTP Client by adding the following code to the `<dependencies>` section of the *pom.xml* file.

   ```xml
   <dependency>
      <groupId>org.apache.httpcomponents</groupId>
      <artifactId>httpclient</artifactId>
      <version>4.5.13</version>
   </dependency>
   ```

1. Add a new rest endpoint called `ssl-test-outbound`. This endpoint opens up a TLS socket to itself and verifies that the TLS connection accepts the TLS/SSL certificate.

   Replace the contents of *SsltestApplication.java* with the following code.

   ```java
   package com.contoso.ssltest;

   import java.security.KeyStore;
   import javax.net.ssl.HostnameVerifier;
   import javax.net.ssl.SSLContext;
   import javax.net.ssl.SSLSession;

   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import com.azure.security.keyvault.jca.KeyVaultLoadStoreParameter;
   import org.springframework.http.HttpStatus;
   import org.springframework.http.ResponseEntity;
   import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;
   import org.springframework.web.client.RestTemplate;

   import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
   import org.apache.http.conn.ssl.TrustSelfSignedStrategy;
   import org.apache.http.impl.client.CloseableHttpClient;
   import org.apache.http.impl.client.HttpClients;
   import org.apache.http.ssl.SSLContexts;

   @SpringBootApplication
   @RestController
   public class SsltestApplication {

       public static void main(String[] args) {
           SpringApplication.run(SsltestApplication.class, args);
       }

       @GetMapping(value = "/ssl-test")
       public String inbound(){
           return "Inbound TLS is working!!";
       }

       @GetMapping(value = "/ssl-test-outbound")
       public String outbound() throws Exception {
           KeyStore azureKeyVaultKeyStore = KeyStore.getInstance("AzureKeyVault");
           KeyVaultLoadStoreParameter parameter = new KeyVaultLoadStoreParameter(
               System.getProperty("azure.keyvault.uri"));
           azureKeyVaultKeyStore.load(parameter);
           SSLContext sslContext = SSLContexts.custom()
                                              .loadTrustMaterial(azureKeyVaultKeyStore, null)
                                              .build();

           HostnameVerifier allowAll = (String hostName, SSLSession session) -> true;
           SSLConnectionSocketFactory csf = new SSLConnectionSocketFactory(sslContext, allowAll);

           CloseableHttpClient httpClient = HttpClients.custom()
               .setSSLSocketFactory(csf)
               .build();

           HttpComponentsClientHttpRequestFactory requestFactory =
               new HttpComponentsClientHttpRequestFactory();

           requestFactory.setHttpClient(httpClient);
           RestTemplate restTemplate = new RestTemplate(requestFactory);
           String sslTest = "https://localhost:8443/ssl-test";

           ResponseEntity<String> response
               = restTemplate.getForEntity(sslTest, String.class);

           return "Outbound TLS " +
               (response.getStatusCode() == HttpStatus.OK ? "is" : "is not")  + " Working!!";
       }

       @GetMapping(value = "/exit")
       public void exit() {
           System.exit(0);
       }

   }
   ```

1. Build the app.

   ```bash
   cd ssltest
   mvn clean package
   ```

1. Upload the app again using the same `sftp` command from earlier in this article.

   ```bash
   cd target
   sftp <your VM public IP address>
   put *.jar
   ```

1. Run the app on the VM.

   ```bash
   set -o noglob
   ssh azureuser@<your VM public IP address> "java -jar *.jar"
   ```

1. After the server is running, verify that the server accepts the TLS/SSL certificate. In the same Bash shell where you issued the previous `curl` command, run the following command.

   ```bash
   curl --insecure https://<your VM public IP address>:8443/ssl-test-outbound
   ```

   You should see the message `Outbound TLS is working!!`.

1. Invoke the `exit` path to kill the server and close the network sockets.

   ```bash
   curl --insecure https://<your VM public IP address>:8443/exit
   ```

You've now observed a simple illustration of the *load*, *present*, and *accept* actions with a self-signed TLS/SSL certificate stored in Azure Key Vault.

## Clean up resources

When you're finished with the Azure resources you created in this tutorial, you can delete them using the following command:

```azurecli
az group delete --name <your resource group name>
```

## Next steps

Explore other things you can do with Spring and Azure.

> [!div class="nextstepaction"]
> [More Spring Boot Starters](spring-boot-starters-for-azure.md)
