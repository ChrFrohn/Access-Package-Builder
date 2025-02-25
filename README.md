# PowerShell Script for Access Package Builder - Creating Access Packages Automatically

First of a big shout out to Nico Wyss for creating the Access Package Builder tool and for inviting me to contribute to this tool!

## Introduction

By leveraging the Access Package Builder tool - https://accesspackagebuilder.azurewebsites.net/builder - you can get recommendations on what access packages could be beneficial to create for your organization. As you may or may not know, access packages in Entra ID Governance provide a great way of bundling resources together, making it easier to assign access to users in your organization. For example, your Human Resources department might have three security groups that provide access to three different applications, each with different permission levels. Instead of manually assigning those three groups to one or more users, you can create an Access Package in Entra ID Governance and add those three security groups to it. You might call this Access Package "Department - HR." This task can be completed very quickly.

However, if you have thousands of groups and hundreds of departments or roles in your company, it can be difficult to determine which groups should be part of which access package. This is where the Access Package Builder comes into play. By leveraging the Access Package Builder, you can get an overview and recommendations on what access packages you should create, and which security groups should be part of each access package. With this visual diagram/map, you are able to quickly determine what access packages you should have.

![image](https://github.com/user-attachments/assets/35945dcf-6518-48db-a71b-cf24ceed6600)


Now that you have an overview and list of access packages and corresponding groups that should be added to each access package, you have two options. You can create them manually yourself, or you can use this PowerShell script to create the access packages with groups automatically. The Access Package Builder can provide you with a JSON file that needs to be used with the PowerShell script.

## The PowerShell script

Please note you will be asked to proceed with creating the access package after you have typed in your information. You will also get a quick overview of what access packages, and which groups will be added to them before the script performs any write operations in your tenant.

### The JSON file

The PowerShell script will ask for the JSON file to begin with. The JSON file needs to be downloaded from here: https://accesspackagebuilder.azurewebsites.net/builder. Then, you can write or paste the path to the JSON file like this: "C:\Users\Frohn\Downloads\access-packages.json." If nothing is inputted, the script will exit.

The JSON file looks something like this:

```json
{
  "defaultAccessPackage": {
    "Default": [ "Group 1", "Group 2" ]
  },
  "companyAccessPackages": {
    "CompanyName": [ "Group 3" ]
  },
  "departmentAccessPackages": {
    "DepartmentName1": [ "Group 4", "Group 5" ],
    "DepartmentName2": [ "Group 6" ]
  }
}
```

### Tenant ID

The next input the PowerShell script needs is your Tenant ID. This is required in order to connect to the tenant where the access packages will be created. Your tenant ID is unique for each tenant and can be found here: https://entra.microsoft.com/#view/Microsoft_AAD_IAM/TenantOverview.ReactView. It should look something like this: 7a0d6d06-8a78-4a32-aac9-23050ad2ba20.

The PowerShell script will check if you input something that matches a tenant ID, and if not, it will exit.

### Catalogs in Entra ID Governance

Each access package needs to be created in a Catalog in Entra ID Governance. This is where the access package along with resources (groups) will be stored and managed. The PowerShell script will create the following three catalogs in Entra ID Governance if they don't already exist (if they do exist, they will be used to add the access packages). The catalogs need to be created before the access packages can be created:

- Catalog 1: Default
- Catalog 2: Company
- Catalog 3: Department

Here's how the script creates catalogs:

```powershell
function New-Catalog {
    param (
        [string]$CatalogDisplayName,
        [string]$CatalogDescription
    )

    try {
        New-MgEntitlementManagementCatalog -DisplayName $CatalogDisplayName -Description $CatalogDescription | out-null
        Write-Host "Catalog '$CatalogDisplayName' created successfully." -ForegroundColor Green
    } catch {
        Write-Host "Failed to create catalog '$CatalogDisplayName'. Error: $_" -ForegroundColor Red
    }
}
```

### Creating the access package

Once the catalogs are created or acquired, the script will proceed to create the access packages based on the information from the JSON file. The access packages will be created in their respective catalogs and nothing more. There won't be any groups or policies added to the access packages yet.

The creation of the access packages in the PowerShell script is handled by the function named "New-AccessPackage."

```powershell
function New-AccessPackage {
    param (
        [string]$DisplayName,
        [string]$Description,
        [string]$CatalogId
    )

    $AccessPackageParameters = @{
        displayName = $DisplayName
        description = $Description
        isHidden = $false
        catalog = @{
            id = $CatalogId
        }
    }
    
    try {
        New-MgEntitlementManagementAccessPackage -BodyParameter $AccessPackageParameters | out-null
        Write-Host "Access package '$DisplayName' created successfully." -ForegroundColor Green
    } catch {
        Write-Host "Failed to create access package '$DisplayName'. Error: $_" -ForegroundColor Red
    }
}
```

### The Access package policy (default)

When you create an access package, you need to have a policy for that access package. The policy is responsible for handling who can request the access package, how long they should have access, and who approves the access package assignment.

The PowerShell script will add a default policy to each access package with various properties.

PLEASE NOTE THE ACCESS PACKAGE POLICY IS DISABLED WHEN CREATED.

```powershell
function New-AccessPackagePolicy {
    param (
        [string]$AccessPackageId
    )

    $params = @{
        displayName = "Initial Policy"
        description = "Initial Policy"
        allowedTargetScope = "notSpecified"
        specificAllowedTargets = @()
        expiration = @{
            endDateTime = $null
            duration = $null
            type = "noExpiration"
        }
        requestorSettings = @{
            enableTargetsToSelfAddAccess = $false
            enableTargetsToSelfUpdateAccess = $false
            enableTargetsToSelfRemoveAccess = $false
            allowCustomAssignmentSchedule = $true
            enableOnBehalfRequestorsToAddAccess = $false
            enableOnBehalfRequestorsToUpdateAccess = $false
            enableOnBehalfRequestorsToRemoveAccess = $false
            onBehalfRequestors = @()
        }
        requestApprovalSettings = @{
            isApprovalRequiredForAdd = $false
            isApprovalRequiredForUpdate = $false
            stages = @()
        }
        accessPackage = @{
            id = $AccessPackageId
        }
    }

    try {
        New-MgEntitlementManagementAssignmentPolicy -BodyParameter $params | out-null
        Write-Host "Access package policy created successfully for Access Package ID: $AccessPackageId." -ForegroundColor Green
    } catch {
        Write-Host "Failed to create access package policy for Access Package ID: $AccessPackageId. Error: $_" -ForegroundColor Red
    }
}
```

### Auto assignment access package Policy

Besides the creation of the default access package policy, an auto-assignment access package policy will also be created for each access package. A policy like this will automatically assign users that fit the criteria of the policy, such as all users with "HR" in their department and located in the city "Copenhagen."

The function named "New-AccessPackageAutoAssignmentPolicy" will create an auto-assignment policy for each access package.

### Adding the groups to the access packages

When the access packages have been created in their respective catalogs and the policies have been created and added to them, the PowerShell script will then proceed to add the groups to the access packages. Before a group can be added to an access package, the group first needs to be added to the catalog as a resource. Once that has been done, the group can then be added to the access package. The function that handles this task will also perform a check on the group, as not all group types can be used with access packages.

The following group types cannot be added to an access package:

- Exchange groups of any kind (Distribution list, mail-enabled security group, dynamic distribution list)
- On-premises Active Directory groups
- Entra ID dynamic groups

The script will perform a "test" on each group to check if it can be added to the access package. If not, the script will skip that group and continue to the next group. The PowerShell script will output the reason in the terminal as to why the group couldn't be added.

### Display the access package

As mentioned in the beginning, the PowerShell script will output what access packages will be created and what groups will be added to each access package. This is done by the function "Show-AccessPackages." This is meant to provide you with an insight into what is about to happen.

```powershell
function Show-AccessPackages {
    param (
        [string]$PackageType,
        [object]$Packages
    )

    foreach ($Package in $Packages.PSObject.Properties) {
        Write-Host "$PackageType access package: $($Package.Name)" -ForegroundColor Cyan
        Write-Host "Groups: $($Package.Value | Out-String)" -ForegroundColor Yellow
    }
}
```

## Putting it all together

All the functionality you have just read about comes together in the final function named "Invoke-AccessPackages." Inside this function, all the other functions are processed with their respective parameters.

The sample for creating all the access packages in the "default" catalog is this:

```powershell
Invoke-AccessPackages -PackageType "Default" -Packages $JSON.defaultAccessPackage -CatalogDescription "Default catalog"
```

For "Company," it is this:

```powershell
Invoke-AccessPackages -PackageType "Company" -Packages $JSON.companyAccessPackages -CatalogDescription "Company catalog"
```

For "Department," it is this:

```powershell
Invoke-AccessPackages -PackageType "Department" -Packages $JSON.departmentAccessPackages -CatalogDescription "Department catalog"
```

Before these three commands are executed, the PowerShell script will need to connect to the Microsoft Graph with the permissions "EntitlementManagement.ReadWrite.All" and "Group.Read.All."

"EntitlementManagement.ReadWrite.All" is needed to create all the access packages, policies, and catalogs.

"Group.Read.All" is needed to read the properties of the groups to determine if they can be added to the catalogs and access packages.

After a successful connection, you will be asked to confirm if you want to proceed with the creation of the access packages. Press "Y" or "N."

- Y = Yes, continue
- N = No, stop

Once confirmed, the PowerShell script will execute and create the access packages one by one. You will get outputs along the way to provide insights into what is happening.

I hope you find this PowerShell script useful for automating the creation of access packages recommended by the Access Package Builder!
# Scripts

- [CreateAccessPackageFromJSON.ps1](https://github.com/ChrFrohn/Access-Package-Builder/blob/main/CreateAccessPackageFromJSON.ps1)

