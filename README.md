# Trusted Signing Demo

This guide walks you through setting up Trusted Signing and signing a demo executable.

Unless otherwise specified, commands are executed from `git-bash` terminal on Windows machine.

Please refer to [Microsoft Learn Trusted Signing](https://learn.microsoft.com/en-us/azure/trusted-signing/) for the latest documentation.

## Prerequisites

1. An active Azure subscription. If you don't have one, you can create a free account at [Azure Free Account](https://azure.microsoft.com/free/).
2. Azure CLI installed. You can download and install it from [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli).

## Set up Trusted Signing resource

Instructions from: [Quickstart: Set up Trusted Signing](https://learn.microsoft.com/en-us/azure/trusted-signing/quickstart?tabs=registerrp-portal%2Caccount-portal%2Corgvalidation%2Ccertificateprofile-portal%2Cdeleteresources-portal)

1. Log into Azure using az cli:

    ```bash
    az login
    ```

1. Register a Trusted Signing resource provider:

    ```bash
    az provider register --namespace "Microsoft.CodeSigning"
    ```

1. Add the extension for Trusted Signing:

    ```bash
    az extension add --name trustedsigning
    ```

1. Export resource name, resource group, location and certificate profile name as environment variables for subsequent commands:

    ```bash
    RG=trusted-signing-rg
    LOCATION=EastUS
    NAME=TrustedSigningDemo
    PROFILE_NAME=CertProfile
    ```

1. Create resource group:

    ```bash
    az group create --name $RG --location $LOCATION
    ```

1. Create TrustedSigning resource:

    ```bash
    az trustedsigning create -n $NAME -l $LOCATION -g $RG --sku Basic
    ```

### Create identity validation request

This instructions are for individual developer. Follow this link for instructions on how to setup for [organization](https://learn.microsoft.com/en-us/azure/trusted-signing/quickstart?tabs=registerrp-cli%2Caccount-cli%2Cindiedevvalidation%2Ccertificateprofile-cli%2Cadeleteresources-cli#create-an-identity-validation-request)

1. In the Azure portal, go to your new Trusted Signing account.
1. Confirm that you're assigned the Trusted Signing Identity Verifier role.
    1. If not assigned, follow instructions [here](https://learn.microsoft.com/en-us/azure/trusted-signing/tutorial-assign-roles#assign-roles):
    1. In the Azure portal, go to your Trusted Signing account. On the resource menu, select Access Control (IAM).
    1. Click Add, and then select Add role assignment
    1. Select "Trusted Signing Identity Verifier" and click Next
    1. Click on "+ Select members" and look up your user by name or email
    1. Click Select
    1. Click "Review + assign"
1. Repeat steps above to assign "Trusted Signing Certificate Profile Signer" role to the user/group allowed to sign artifacts.
1. On the Trusted Signing account Overview pane or on the resource menu under Objects, select Identity validations.
1. Select Organization, in the dropdown select Individual and then select Public.
1. On New identity validation, provide the following information:
    * First Name
    * Last Name
    * Primary Email
    * Street, City, Country, State, Postal code
1. Select Certificate subject preview to see the preview of the information that appears in the certificate.
1. Select I accept Microsoft terms of use for trusted signing services. You can download the Terms of Use to review or save them.
1. Select the Create button.
1. When the request is successfully created, the identity validation request status changes to In Progress.
1. When the status changes to Action Required. Click on your name, a blade opens on the right-hand side. Click on the link under "Please complete your verification here".
1. Follow the link to complete the Identity Validation process. Use the email address provided at the time of request creation to create a Microsoft account. Enter the credentials when prompted, and you'll be navigated to the next screen.
1. Click on Start under our Trusted Partner > Au10tix to begin the validation process. You will be navigated to a 3rd party website.
1. You need to switch to your mobile device to complete the process and present the relevant documentation when prompted.
1. On your mobile device, open the Authenticator app, select Verified IDs, on bottom right you’ll see the QR code in blue. Click on that.
1. In Azure portal, click on the link that you used to perform identity validation, scan the QR code under Present Verified ID from your mobile device, this completes the process. For successful completion it says: Verification Successful
1. It takes a couple of minutes for the Identity Validation status on Azure portal to update. For a successful Verified ID the status on Azure portal changes to Completed.

### Create a certification profile

For most up to date instructions refer to this [documentation](https://learn.microsoft.com/en-us/azure/trusted-signing/quickstart?tabs=registerrp-portal%2Caccount-portal%2Corgvalidation%2Ccertificateprofile-cli%2Cdeleteresources-portal#create-a-certificate-profile)

Retrieve Identity Validation Id:

1. In the Azure portal, go to your Trusted Signing account.
1. On the Trusted Signing account Overview pane or on the resource menu under Objects, select Identity validations.
1. Select the hyperlink for the relevant entity. On the Identity validation pane, you can copy the value for Identity validation Id.
1. Paste the value into an environment variable:

    ```bash
    VALIDATION_ID=<copied value>
    ```

Create a certificate profile by using Azure CLI:

```bash
az trustedsigning certificate-profile create -g $RG --a $NAME \
-n $PROFILE_NAME --profile-type PublicTrust --identity-validation-id $VALIDATION_ID
```

## Set up SignTool to use Trusted Signing

For latest instructions refer to this [documentation](https://learn.microsoft.com/en-us/azure/trusted-signing/how-to-signing-integrations#set-up-signtool-to-use-trusted-signing).

1. Install Trusted Signing Client Tools using `winget`. For additional installation options see [here](https://learn.microsoft.com/en-us/azure/trusted-signing/how-to-signing-integrations#trusted-signing-client-tools-installer)

    ```bash
    winget install -e --id Microsoft.Azure.TrustedSigningClientTools
    ```

1. Download and install SignTool using nuget

    1. Download nuget if not installed already:

    ```bash
    curl -o nuget.exe https://dist.nuget.org/win-x86-commandline/latest/nuget.exe
    ```

    1. Download and extract Windows SDK Build Tools NuGet package by running the following installation command:

    ```bash
    ./nuget install Microsoft.Windows.SDK.BuildTools -Version 10.0.22621.3233 -x
    ```

1. If not already available, download and install .NET 8.0 Runtime using [Windows x64 installer](https://dotnet.microsoft.com/download/dotnet/thank-you/runtime-8.0.4-windows-x64-installer)

1. Download and install the Trusted Signing dlib package using nuget:

    ```bash
    ./nuget.exe install Microsoft.Trusted.Signing.Client -Version 1.0.53 -x
    ```

1. Create a metadata JSON file

    1. Create a new JSON file: `metadata.json`
    1. Export endpoint URI for the region of your Trusted Signed account:

        ```bash
        ENDPOINT_URI=https://eus.codesigning.azure.net
        ```

    1. Create a new JSON file: `metadata.json` with the values below:

        ```bash
        CAT <<EOF > metadata.json
        {
            "Endpoint": "$ENDPOINT_URI",
            "CodeSigningAccountName": "$NAME",
            "CertificateProfileName": "$PROFILE_NAME"
        }
        EOF
        ```

### Compile .NET demo application (optional)

A simple C# Console application is provided in this repo to allow you to test code signing.
You can test using a different executable, this is only provided here for convenience.

Compile .NET application as follows:

```bash
cd dotnetapp
dotnet build
cd ..
```

### Use SignTool to sign executable file

Use the following command to sign the `dotnetapp.exe` compiled above:

```bash
Microsoft.Windows.SDK.BuildTools/bin/10.0.22621.0/x64/signtool sign \
//v //debug //fd SHA256 //tr http://timestamp.acs.microsoft.com //td SHA256 \
//dlib Microsoft.Trusted.Signing.Client/bin/x64/Azure.CodeSigning.Dlib.dll \
//dmdf metadata.json dotnetapp/bin/Debug/net9.0/dotnetapp.exe
```

If successful you should see output similar to this:

```txt
Submitting digest for signing...

OperationId 65385c73-0834-4d7c-bd3c-ec96dc5457e9: InProgress

Signing completed with status 'Succeeded' in 11.5819863s

Successfully signed: dotnetapp/bin/Debug/net9.0/dotnetapp.exe

Number of files successfully Signed: 1
Number of warnings: 0
Number of errors: 0
```

To verify signature the following command can be used:

```bash
Microsoft.Windows.SDK.BuildTools/bin/10.0.22621.0/x64/signtool verify \
//pa dotnetapp/bin/Debug/net9.0/dotnetapp.exe
```

### Compile Java demo application (optional)

Download and install [JDK 17 or greater](https://learn.microsoft.com/en-us/java/openjdk/download).

Export `JAVA_HOME` variable (your JDK path may differ):

```bash
export JAVA_HOME='/c/Program Files/Eclipse Adoptium/jdk-21.0.2.13-hotspot'
```

A simple Java Spring Boot application is provided in this repo to allow you to test signing JAR files.
You can test using a different JAR file, this is only provided here for convenience.

Compile Java application as follows:

```bash
cd javademoapp
./mvnw package -DskipTests
cd ..
```

### Use jsign and Trusted Signing to sign executable file

**NOTE:** [jsign](https://ebourg.github.io/jsign/) is a third party tool not supported by Microsoft.

1. Download all-in-one `jsign` jar:

    ```bash
    curl -o jsign-7.1.jar https://github.com/ebourg/jsign/releases/download/7.1/jsign-7.1.jar
    ```

1. Get access token for code signing:

    ```bash
    ACCESS_TOKEN=$(az account get-access-token --resource https://codesigning.azure.net \
    --query accessToken --output tsv)
    ```

1. Use command to sign executable:

    ```bash
    java -jar jsign-7.1.jar --storetype TRUSTEDSIGNING \
        --keystore $ENDPOINT_URI \
        --storepass $ACCESS_TOKEN \
        --alias $NAME/$PROFILE_NAME dotnetapp/bin/Debug/net9.0/dotnetapp.exe
    ```

1. Optionally, use `signtool` to verify signature the following command can be used:

```bash
Microsoft.Windows.SDK.BuildTools/bin/10.0.22621.0/x64/signtool verify \
//pa dotnetapp/bin/Debug/net9.0/dotnetapp.exe
```

#### Use jarsigner to sign a jar file

`jsign` does not directly support signing `jar` files, however this can be accomplished in conjunction with `jarsigner`, a program included in the JRE (Java Runtime Environment).

As a prerequisite, you first need to setup JRE keystore to trust Microsoft Root Certificate as explained in this [Github issue](https://github.com/ebourg/jsign/discussions/268).

1. Download [Microsoft Identity Verification Root Certificate Authority 2020.crt](http://www.microsoft.com/pkiops/certs/Microsoft%20Identity%20Verification%20Root%20Certificate%20Authority%202020.crt)

    ```bash
    curl -o 'Microsoft Identity Verification Root Certificate Authority 2020.crt' \
    http://www.microsoft.com/pkiops/certs/Microsoft%20Identity%20Verification%20Root%20Certificate%20Authority%202020.crt
    ```

1. Import Microsoft Root CA certificate into your JRE's `cacerts` file:

    ```bash
    sudo keytool -importcert -alias microsoft-identity-root-ca \
    -keystore "$JAVA_HOME/lib/security/cacerts" -storepass changeit \
    -file "Microsoft Identity Verification Root Certificate Authority 2020.crt"
    ```

1. Provide sudo password if prompted and enter **yes** when prompted to trust certificate.

1. Sign the jar file using the command below:

    ```bash
    jarsigner -J-cp -Jjsign-7.1.jar -J--add-modules -Jjava.sql \
    -providerClass net.jsign.jca.JsignJcaProvider \
    -providerArg $ENDPOINT_URI -keystore NONE \
    -storetype TRUSTEDSIGNING -storepass $ACCESS_TOKEN \
    javademoapp/target/demo-0.0.1-SNAPSHOT.jar $NAME/$PROFILE_NAME 
    ```

1. Optionally to verify the jar file has been signed, use this command:

    ```bash
    jarsigner -verify -verbose -certs javademoapp/target/demo-0.0.1-SNAPSHOT.jar
    ```

1. As a final verification step, you can optionally attempt to run the `jar` and confirm it is still functional:

    ```bash
    cd javademoapp
    ./mvnw spring-boot:run
    ```
