# Appendix

## Tray Debug

To open the tray debug, click **STRG + SHIFT + Click on the RealmJoin Icon**. The Client menu will open but with a further entry at the end of the menu: **Show Debug Window**.

Client menu:

[![RJclientmenu2](.gitbook/assets/rj-ui4.png)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-ui4.png)

Tray debug:

[![RJtraydebug](.gitbook/assets/rj-ui5.png)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-ui5.png)

Show Debug Window contains seven different diagnostic tools. If a device is not able to be addressed by the server or can not connect to the back-end, this tool will provide the user with the tools for the first steps of diagnosis.

## Protocol Handler

It is possible to install RealmJoin packages using an URL-link.  
The correct format for this command consists of the `realmjoin:` call, the subcommand `install` and the package ID, e.g. `generic-google-chrome`.  
The complete link therefore would be written as:  
`realmjoin:install:generic-google-chrome`.  
The package has to be assigned for the user in the RealmJoin back-end.

## NoGraph Option

To install RealmJoin without Graph API consent, the registry key

```text
Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\realmjoin\Parameters\NoGraph
```

can be set to `1`.  
It is also possible to set this key during the installation of RealmJoin as a argument for the **msi**:

`msiexec /i "RealmJoin.msi" NOGRAPH=1`.

## 3rd Party NuGet Packages

[https://packages.gkdatacenter.net/](https://packages.gkdatacenter.net/)

### PowerShell Modules

#### PSIni

* Used to read and write ini files from within Chocolatey or craft packages
* Source & License: [https://www.powershellgallery.com/packages/PsIni](https://www.powershellgallery.com/packages/PsIni)
* Project Home: [http://lipkau.github.io/PsIni/](http://lipkau.github.io/PsIni/)

Use: Deploy as package through RealmJoin \(or add dependency in nuspec, e.g.: `<dependency id="PSini" version="[2.0,3.0)" />`\)  
 Import: `Import-Module "$env:ProgramData\chocolatey\lib\PSini\PsIni.psm1"`

## RealmJoin Workflow

The RealmJoin workflow describes the detailed process after RealmJoin is deployed via Intune MDM Channel \(single MSI software deployment\) to the client.

To understand how RealmJoin detects and authenticates the device and the related user we must investigate the Azure AD/Intune device enrollment process first:

1. After the user successfully authenticates against Azure AD by providing username, password and MFA, the **Azure AD Device Registration Service** generates a key pair for the device certificate.
2. Generates a **Certificate Signing Request** \(CSR\) using the key pair. Signs CSR data with private key plus public key in request.
3. Generates a second key pair that will be used to bind SSO tokens physically to the device when authenticating to Azure AD later on. This key is typically called **storage/transport key** \(Kstk\) and is derived from **Storage Root Key** \(SRK\) of the device **Trusted Platform Module** \(TPM\). The way the binding of the SSO token to the device is achieved by storing into the TPM a corresponding symmetric session key \(encrypted to Kstk\) issued along with the SSO token upon authentication to Azure AD.
4. Send a device registration request to **Azure DRS** passing along the ID token, the generated CSR and the public portion of the Kstk along with its attestation data.

Once the request comes to Azure DRS the service will validate the token, will create a corresponding device object in Azure AD and will generate and send back a certificate to the device. The API in turn will install the certificate into the **LocalMachine\mystore**

To check the related cert please use the following command:

dir Cert:\LocalMachine\My \| where { $\_.Issuer -match "CN=MS-Organization-Access" } \| fl

[![RJ - Related Cert](.gitbook/assets/rj-workflow1.png)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-workflow1.png)

The related **Azure AD Device ID** can be checked here:

[![RJ - Check related AAD ID](.gitbook/assets/rj-workflow2.png)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-workflow2.png)

Having this information available out of the related certificate, RealmJoin is now able to start the provisioning process without the need of a dedicated user authentication/interaction. RealmJoin knows about the Azure AD Device ID out of the above described device certificate information and can start all processes running within the system context for this device and the corresponding user account.

[![RJ - Certificate Information](.gitbook/assets/rj-workflow3.png)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-workflow3.png)

### Initial run

As RealmJoin is aware about the Azure AD Device ID and the related user account, it will immediately start \(after the regular OOBE process finished\) to apply the device checks and install the users' software within the system context of the device. After completion of this initial run, RealmJoin will trigger a device restart.

After a device restart RealmJoin will prompt the user for an authentication confirmation - no credentials like password needed here! Now RealmJoin can proceed with all user based device and software configuration settings.

1. RealmJoin will ask the user for its **Second Identity** - this feature is only available when a **Legacy Active Directory Authentication Provider** should be involved into the users' network resource scope by providing **NTML tokens** for on-prem file or print services.
2. The build in RealmJoin "Security Requirements" assessment does some **pre-checks**:
3. System Update Status: not enforced
4. Full-Disk Encryption: enforced
5. Firewall Configuration: enforced
6. Anti-Virus Configuration: not enforced  

> \[!NOTE\] These checks are customizable \(enforce/not enforce\) and will be replaced by the **Intune Device Health Check** in future.

During the initial run of RealmJoin, the **BitLocker Drive Encryption** will be enabled and enforced. User interactivity is not necessary. The related BitLocker recovery key is escrowed into Azure AD.

> \[!NOTE\] To find this key is required, please use this URL \(admin only\)

## Initial Start before RJ v4.15

When RealmJoin is enrolled and started for the first time, it asks for the user identity and then calls to the cloud service for a policy.

[![RJ AAD Auth](.gitbook/assets/rj-aad-auth.png)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-aad-auth.png)

RealmJoin **Security Requirement** assessment does some pre-checks \(Encryption, Patch Level, Firewall, Anti-Virus, etc. – this is optional and can be replaced in parts by Intune-Health-Check\). In the last step, all mandatory software will be installed \(**black screen installation**\). During this installation, any interaction with the client is suppressed.

[![RJ Sec Check](.gitbook/assets/rj-sec-check.gif)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-sec-check.gif)

If no error occurs during deployment, RealmJoin is ready to use.

### Client usage

After being successfully installed, RealmJoin is automatically started on the user login and is permanent active in the background. It is represented with an ID card icon. Clicking on the icon opens up the RealmJoin client menu.  
It contains basic information in the lower and a number of links in the upper part. The selector **Software Packages** opens a second context menu with all the software packages that are allocated to the user.

[![RJ Tray](.gitbook/assets/rj-tray-menu%20%281%29.png)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-tray-menu.png)

If an user wishes to install any of the listed software, he/she is only required to select the package to start the installation.

[![RJ Add Package](.gitbook/assets/rj-client-addpackage2.png)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-client-addpackage2.png)

The installation mode depends on the packages selected: If those are only user mode packages, they are installed immediately. In case of a higher permission level, RealmJoin starts a service \(realmjoinservice.exe\) and installs the packages with the **SYSTEM** user account.

### Debug mode

To open a debug window, press and hold **Shift** + **Strg** and click the RealmJoin icon. This reveals **Show Debug Window** as a further entry at the end of the context menu. Show Debug Window contains seven different diagnostic tools.  
If a device is not able to be addressed by the server or can not connect to the back-end, this tool will provide the user with the tools for the first steps of diagnosis.

Beside Show Debug Window **Retry base installation** is a further entry that reveals. Retry base installation allows an user to reinstall the RealmJoin client.  
Additionally, when the client tray menu is opened in debug mode, all software packages are shown with a package version number.

[![RJ Debug Menu](.gitbook/assets/rj-debug-menu.png)](https://github.com/realmjoin/realmjoin-gitbooks/tree/3c2250fcc0d712e1b40ac535a1766b57ce01910c/docs/media/rj-debug-menu.png)

**Collect Logs** is a quick way to access all log files, which will be saved in a zip-file to the users desktop. See chapter [Troublehooting](troubleshooting.md) for a detailed description of the RealmJoin debug window and its features.

