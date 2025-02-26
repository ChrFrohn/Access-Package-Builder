# Access Package Builder: Automating Access Package Creation with PowerShell

First of all, a big shout out to (Nico Wyss)[https://www.linkedin.com/in/nico-wyss/] for creating the (Access Package Builder tool)[https://accesspackagebuilder.azurewebsites.net/] and for inviting me to contribute to this project!

## Introduction

Managing access rights in large organizations can be a complex and time-consuming task. If you've worked with Microsoft Entra ID Governance, you probably know about Access Packages - a great way to bundle resources together for easier assignment. But here's the challenge: how do you decide which resources should go into which packages when you have thousands of security groups and hundreds of departments?

This is where the [Access Package Builder](https://accesspackagebuilder.azurewebsites.net/builder) comes in. It analyzes your existing setup and gives you recommendations on what access packages would be beneficial to create, along with which security groups should be added to each package.

![Access Package Builder visualization](https://github.com/user-attachments/assets/35945dcf-6518-48db-a71b-cf24ceed6600)

## A Real-World Example

Let's say your HR department uses three different applications, each requiring different permission levels through security groups. Instead of manually assigning those three groups to every HR employee, you can create an "HR Department" Access Package that bundles them together. When someone joins HR, they get all the necessary access in one assignment.

Now scale that up to an enterprise with dozens of departments and roles. How do you determine the right packages? The Access Package Builder creates a visual diagram/map that helps you quickly identify what packages make sense for your organization.

## Automating Package Creation

Once you have your recommendations, you have two options:
1. Create each package manually (potentially very time-consuming)
2. Use my PowerShell script to automate the entire process

The script takes the JSON output from the Access Package Builder and automatically creates all the recommended packages in your tenant. What might take days of manual configuration can be done in minutes!

## Using the PowerShell Script - Step by Step

### What You'll Need:
- The JSON file from the Access Package Builder containing the recommendations
- Your Tenant ID (found in the Microsoft Entra admin center)
- Basic familiarity with running PowerShell scripts

### The Process:

#### 1. Get Your JSON File
First, visit the [Access Package Builder](https://accesspackagebuilder.azurewebsites.net/builder) and download your recommendations as a JSON file. It will look something like this:

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

#### 2. Run the Script
The script will first ask for the path to your JSON file (e.g., "C:\Users\YourName\Downloads\access-packages.json"). If you don't provide this, the script will exit.

Next, it will ask for your Tenant ID. This is required to connect to the right tenant where the access packages will be created. Your Tenant ID can be found at https://entra.microsoft.com/#view/Microsoft_AAD_IAM/TenantOverview.ReactView and looks something like: 4b0c7e06-9a48-4c32-dab9-44054bd2ba22.

#### 3. Review and Confirm
Before making any changes, the script will show you what it's going to create:
- Which access packages will be created
- Which security groups will be added to each package

You'll be asked to confirm with 'Y' (Yes, continue) or 'N' (No, stop) before any changes are made.

#### 4. Automatic Creation
After confirmation, the script will:
- Create three catalogs if they don't already exist:
  - Default
  - Company
  - Department
- Create the recommended access packages in these catalogs
- Set up default policies (which control how packages can be assigned)
- Set up auto-assignment policies (which can automatically assign packages based on user attributes)
- Add all compatible security groups to each package

## Technical Details

### Catalog Creation
Each access package needs to be created within a Catalog. The script creates or uses three catalogs:

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

```powershell
function Get-OrCreateCatalog {
    param (
        [string]$CatalogDisplayName,
        [string]$CatalogDescription
    )

    $Catalogs = Get-MgEntitlementManagementCatalog -All
    $Catalog = $Catalogs | Where-Object DisplayName -eq $CatalogDisplayName
    if ($Catalog) {
        Write-Host "Catalog '$CatalogDisplayName' already exists." -ForegroundColor Yellow
    } else {
        try {
            New-Catalog -CatalogDisplayName $CatalogDisplayName -CatalogDescription $CatalogDescription
            $Catalog = Get-MgEntitlementManagementCatalog -Filter "displayName eq '$CatalogDisplayName'"
        } catch {
            Write-Host "Failed to create catalog '$CatalogDisplayName'. Error: $_" -ForegroundColor Red
            return $null
        }
    }
    return $Catalog
}
```

### Access Package Creation
The script creates access packages based on the JSON file:

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

### Default Policy Creation
Each access package needs a policy to control who can request it and how. Note that these policies are disabled by default:

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

### Auto-Assignment Policy Creation
The script also creates auto-assignment policies that can automatically assign packages based on user attributes:

```powershell
function New-AccessPackageAutoAssignmentPolicy {
    param (
        [string]$PolicyName,
        [string]$PolicyDescription,
        [string]$AccessPackageId,
        [string]$AutoAssignmentPolicyFilter
    )

    $AutoPolicyParameters = @{
        displayName = $PolicyName
        description = $PolicyDescription
        allowedTargetScope = "specificDirectoryUsers"
        specificAllowedTargets = @(
            @{
                "@odata.type" = "#microsoft.graph.attributeRuleMembers"
                description = $PolicyDescription
                membershipRule = $AutoAssignmentPolicyFilter
            }
        )
        accessPackage = @{
            id = $AccessPackageId
        }
    }

    try {
        New-MgEntitlementManagementAssignmentPolicy -BodyParameter $AutoPolicyParameters | out-null
        Write-Host "Access package assignment policy: '$PolicyName' created successfully." -ForegroundColor Green
    } catch {
        Write-Host "Failed to create access package assignment policy '$PolicyName'. Error: $_" -ForegroundColor Red
    }
}
```

### Adding Groups to Packages
The script performs compatibility checks before adding groups to packages. Not all group types can be used with access packages:

- Exchange groups (Distribution lists, mail-enabled security groups, dynamic distribution lists)
- On-premises Active Directory groups
- Entra ID dynamic groups

The script automatically identifies and skips incompatible groups:

```powershell
Function Add-EntraGroupToAccessPackage {
    param (
        [string]$CatalogId,
        [string]$GroupName,
        [string]$AccessPackageId
    )

    # Get the Group from Entra
    try {
        $EntraGroup = Get-MgGroup -Filter "DisplayName eq '$GroupName'"
    }
    catch {
        Write-Host "Error finding group '$GroupName': $_" -ForegroundColor Red
        return
    }

    if ($EntraGroup -eq $null) {
        Write-Host "Group '$GroupName' not found." -ForegroundColor Red
        return
    }

    $EntraGroup = $EntraGroup | Where-Object {
        ($_.ProxyAddresses.Count -eq 0) -or
        ($_.OnPremisesSyncEnabled -eq $false) -and
        ($_.GroupTypes -notcontains "DynamicMembership")
    }

    if ($EntraGroup) {
        Write-Host "Group found: $($EntraGroup.DisplayName)" -ForegroundColor Green
        $GroupObjectId = $EntraGroup.Id

        # Check if the group is already a resource in the catalog
        $CatalogResources = Get-MgEntitlementManagementCatalogResource -AccessPackageCatalogId $CatalogId -All
        $GroupResource = $CatalogResources | Where-Object { $_.OriginId -eq $GroupObjectId }

        if ($GroupResource) {
            Write-Host "The group with ID '$GroupObjectId' is already a resource in the catalog." -ForegroundColor Yellow
        } else {
            # Add the Group as a resource to the Catalog
            $GroupResourceAddParameters = @{
                requestType = "adminAdd"
                resource = @{
                    originId = $GroupObjectId
                    originSystem = "AadGroup"
                }
                catalog = @{
                    id = $CatalogId
                }
            }

            try {
                New-MgEntitlementManagementResourceRequest -BodyParameter $GroupResourceAddParameters | Out-Null
                Write-Host "Group with ID '$GroupObjectId' added to catalog successfully." -ForegroundColor Green
            } catch {
                Write-Host "Failed to add group with ID '$GroupObjectId' to catalog. Error: $_" -ForegroundColor Red
                return
            }
        }

        # Get the Group as a resource from the Catalog
        $CatalogResources = Get-MgEntitlementManagementCatalogResource -AccessPackageCatalogId $CatalogId -ExpandProperty "scopes" -All
        $GroupResource = $CatalogResources | Where-Object { $_.OriginId -eq $GroupObjectId }
        $GroupResourceScope = $GroupResource.Scopes[0]

        # Add the Group as a resource role to the Access Package
        $GroupResourceFilter = "(originSystem eq 'AadGroup' and resource/id eq '$($GroupResource.Id)')"
        $GroupResourceRoles = Get-MgEntitlementManagementCatalogResourceRole -AccessPackageCatalogId $CatalogId -Filter $GroupResourceFilter -ExpandProperty "resource"
        $GroupMemberRole = $GroupResourceRoles | Where-Object { $_.DisplayName -eq "Member" }

        $GroupResourceRoleScopeParameters = @{
            role = @{
                displayName = "Member"
                description = ""
                originSystem = $GroupMemberRole.OriginSystem
                originId = $GroupMemberRole.OriginId
                resource = @{
                    id = $GroupResource.Id
                    originId = $GroupResource.OriginId
                    originSystem = $GroupResource.OriginSystem
                }
            }
            scope = @{
                id = $GroupResourceScope.Id
                originId = $GroupResourceScope.OriginId
                originSystem = $GroupResourceScope.OriginSystem
            }
        }

        try {
            New-MgEntitlementManagementAccessPackageResourceRoleScope -AccessPackageId $AccessPackageId -BodyParameter $GroupResourceRoleScopeParameters | Out-Null
            Write-Host "Group '$GroupName' added to access package successfully." -ForegroundColor Green
        } catch {
            Write-Host "Failed to add group '$GroupName' to access package. Error: $_" -ForegroundColor Red
        }
    } else {
        Write-Host "Group '$GroupName' does not meet criteria." -ForegroundColor Yellow
    }
}
```

## Putting It All Together

All of these functions come together in the main `Invoke-AccessPackages` function:

```powershell
function Invoke-AccessPackages {
    param (
        [string]$PackageType,
        [object]$Packages,
        [string]$CatalogDescription
    )

    Write-Host "Processing $PackageType access packages..." -ForegroundColor Magenta
    $Catalog = Get-OrCreateCatalog -CatalogDisplayName $PackageType -CatalogDescription $CatalogDescription
    if ($null -eq $Catalog) {
        return
    }

    foreach ($Package in $Packages.PSObject.Properties) {
        Write-Host "Processing access package: $($Package.Name)" -ForegroundColor Magenta
        # Check if the access package already exists
        $ExistingAccessPackage = Get-MgEntitlementManagementAccessPackage -Filter "displayName eq '$($Package.Name)'"
        if ($ExistingAccessPackage) {
            Write-Host "Access package '$($Package.Name)' already exists." -ForegroundColor Yellow
            $GetTheNewAccessPackage = $ExistingAccessPackage
        } else {
            try {
                New-AccessPackage -DisplayName $Package.Name -Description "$PackageType access package: $($Package.Name)" -CatalogId $Catalog.id
                $GetTheNewAccessPackage = Get-MgEntitlementManagementAccessPackage -Filter "displayName eq '$($Package.Name)'"

                # Create the "Initial Policy" for the access package
                New-AccessPackagePolicy -AccessPackageId $GetTheNewAccessPackage.Id

                # Create access package auto assignment policy for the access package
                $AutoAssignmentPolicyFilter = "(user.$PackageType -eq `"$($Package.Name)`")"
                New-AccessPackageAutoAssignmentPolicy -PolicyName "Policy for $($Package.Name)" -PolicyDescription "Policy for $($Package.Name)" -AccessPackageId $GetTheNewAccessPackage.Id -AutoAssignmentPolicyFilter $AutoAssignmentPolicyFilter
            } catch {
                Write-Host "Failed to create access package '$($Package.Name)'. Error: $_" -ForegroundColor Red
                continue
            }
        }

        # Add the groups to the access package
        foreach ($EntraGroup in $Package.Value) {
            Write-Host "Adding group '$EntraGroup' to catalog and access package..." -ForegroundColor Magenta
            Add-EntraGroupToAccessPackage -CatalogId $Catalog.Id -GroupName $EntraGroup -AccessPackageId $GetTheNewAccessPackage.Id
        }
    }
    Write-Host "Finished processing $PackageType access packages." -ForegroundColor Green
}
```

## Permissions Required

The script requires two Microsoft Graph permissions:
- `EntitlementManagement.ReadWrite.All` (to create catalogs, packages, and policies)
- `Group.Read.All` (to read group properties and determine compatibility)

## The Result

After running the script, you'll have a well-organized set of access packages that align with your organizational structure. What might have taken days of manual configuration can be done in minutes!

I hope you find this PowerShell script useful for automating the creation of access packages recommended by the Access Package Builder. If you have any questions or feedback, please feel free to reach out.

## Script Repository

You can find the full script here:
- [CreateAccessPackageFromJSON.ps1](https://github.com/ChrFrohn/Access-Package-Builder/blob/main/CreateAccessPackageFromJSON.ps1)