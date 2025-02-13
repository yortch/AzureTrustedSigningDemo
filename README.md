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

1. Export resource name, resource group, location as environment variables for subsequent commands:

    ```bash
    RG=trusted-signing-rg
    LOCATION=EastUS
    NAME=TrustedSigningDemo
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
-n CertProfile --profile-type PublicTrust --identity-validation-id $VALIDATION_ID
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
            "CertificateProfileName": "CertProfile"
        }
        EOF
        ```

### Compile demo application (optional)

A simple C# Console application is provided in this repo to allow you to test code signing.
You can test using a different executable, this is only provided here for convenience.

Compile .NET application as follows:

```bash
cd dotnetapp
dotnet build
```

### Use SignTool to sign a file

Use the following command to sign the `dotnetapp.exe` compiled above:

```bash
Microsoft.Windows.SDK.BuildTools/bin/10.0.22621.0/x64/signtool sign //v //debug //fd SHA256 //tr http://timestamp.acs.microsoft.com //td SHA256 //dlib Microsoft.Trusted.Signing.Client/bin/x64/Azure.CodeSigning.Dlib.dll //dmdf metadata.json dotnetapp/bin/Debug/net9.0/dotnetapp.exe
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
