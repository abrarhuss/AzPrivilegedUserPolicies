# Azure Policy: Deny High-Privilege Role Assignments with PIM-Awareness

## Overview

This Azure Policy definition enforces least-privilege access by preventing assignment of high-privilege roles through both direct assignments and Privileged Identity Management (PIM) operations, while supporting breakglass exceptions.

## Key Features

- 🛡️ **Comprehensive Protection**: Blocks both direct role assignments and PIM-related operations
- 🔒 **PIM-Aware**: Evaluates PIM schedule/eligibility resources to prevent circumvention
- 🚨 **Breakglass Support**: Allows designated emergency access accounts
- ⚙️ **Configurable**: Parametrized design for flexible deployment
- 📊 **Audit Mode**: Supports evaluation before enforcement

## Policy Details

### Resource Types Covered
- `Microsoft.Authorization/roleAssignments`
- `Microsoft.Authorization/roleEligibilityScheduleRequests`
- `Microsoft.Authorization/roleAssignmentScheduleRequests`

### Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|----------|
| `roleDefinitionIds` | Array | Role definitions to block | Owner role (`8e3af657-a8ff-443c-a75c-2fe8c4bcb635`) |
| `excludedPrincipalIds` | Array | Break-glass account Object IDs | `[]` |
| `effect` | String | Policy effect (deny/audit/disabled) | `deny` |

## Deployment Guide

### 1. Azure CLI

```bash
# Create policy definition
az policy definition create \
  --name "Deny-High-Privilege-Roles-PIM-Aware" \
  --display-name "Deny High-Privilege Role Assignments (PIM-Aware)" \
  --rules ./deny-high-privilege-roles-parametrized-with-breakglass-exception-pim-aware.json \
  --mode All

# Assign policy (example)
az policy assignment create \
  --name "DenyHighPrivRoles-Assignment" \
  --scope /subscriptions/<subscriptionId> \
  --policy "Deny-High-Privilege-Roles-PIM-Aware" \
  --params "{ \"roleDefinitionIds\": { \"value\": [\"/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635\"] }, \"excludedPrincipalIds\": { \"value\": [\"<breakglass-objectId>\"] }, \"effect\": { \"value\": \"deny\" } }"
```

### 2. PowerShell

```powershell
# Create policy definition
New-AzPolicyDefinition -Name "Deny-High-Privilege-Roles-PIM-Aware" `
  -DisplayName "Deny High-Privilege Role Assignments (PIM-Aware)" `
  -Policy ./deny-high-privilege-roles-parametrized-with-breakglass-exception-pim-aware.json

# Assign policy (example)
$params = @{
    "roleDefinitionIds" = @{ "value" = @("/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635") }
    "excludedPrincipalIds" = @{ "value" = @("<breakglass-objectId>") }
    "effect" = @{ "value" = "deny" }
}

New-AzPolicyAssignment -Name "DenyHighPrivRoles-Assignment" `
  -Scope "/subscriptions/<subscriptionId>" `
  -PolicyDefinition (Get-AzPolicyDefinition -Name "Deny-High-Privilege-Roles-PIM-Aware") `
  -PolicyParameterObject $params
```

## Implementation Steps

1. **Preparation**
   - Identify high-privilege roles to restrict
   - List breakglass account Object IDs
   - Plan deployment scope

2. **Testing Phase**
   - Deploy with `audit` effect
   - Test role assignments
   - Review compliance state
   - Validate breakglass exceptions

3. **Production Deployment**
   - Update effect to `deny`
   - Monitor policy evaluations
   - Configure alerts

4. **Maintenance**
   - Regular review of breakglass accounts
   - Audit policy effectiveness
   - Update role definitions as needed

## Security Considerations

### Best Practices
- Maintain minimum two breakglass accounts
- Rotate breakglass credentials regularly
- Monitor breakglass account usage
- Review excluded principals periodically
- Combine with PIM approval workflows

### Limitations
- Requires exact role definition IDs
- No automatic remediation of existing assignments
- Future PIM resource types may need policy updates

## Monitoring & Compliance

### Recommended Monitoring
- Azure Policy compliance state
- Activity log for policy evaluations
- PIM activation requests
- Breakglass account usage

### Alert Setup
- Policy state changes
- Denied assignment attempts
- Breakglass account activations

## Troubleshooting

### Common Issues
1. **Policy Not Blocking**
   - Verify principal not excluded
   - Check role definition ID format
   - Confirm policy scope

2. **Breakglass Access Failed**
   - Validate Object ID
   - Check policy assignment
   - Verify PIM settings

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2023-10-23 | Initial release |
| 1.1.0 | 2023-10-23 | Added PIM-aware controls |

## Resources

- [Azure RBAC Documentation](https://learn.microsoft.com/azure/role-based-access-control/overview)
- [Azure PIM Documentation](https://learn.microsoft.com/azure/active-directory/privileged-identity-management/pim-configure)
- [Azure Policy Overview](https://learn.microsoft.com/azure/governance/policy/overview)

## License

MIT License - See repository root for full license text.

## Contributing

Contributions welcome! Please submit issues and pull requests.

---

**Author**: Abrar Hussain  
**Repository**: [abrarhuss/AzPrivilegedUserPolicies](https://github.com/abrarhuss/AzPrivilegedUserPolicies)  
**File**: deny-high-privilege-roles-parametrized-with-breakglass-exception-pim-aware.json

### Helper Commands for Policy Parameters

#### List Role Definitions
These PowerShell commands help identify role definitions to include in the `roleDefinitionIds` parameter:

```powershell
# List all built-in roles with their IDs
Get-AzRoleDefinition | Where-Object IsCustom -eq $false | Select-Object Name, Id | Sort-Object Name

# Find specific high-privilege roles
$highPrivilegeRoles = @(
    'Owner',
    'Contributor',
    'User Access Administrator',
    'Co-Administrator'
)

Get-AzRoleDefinition | Where-Object { 
    $_.Name -in $highPrivilegeRoles 
} | Select-Object Name, Id | Format-Table -AutoSize

# Export high-privilege roles to CSV
$highPrivRoles = Get-AzRoleDefinition | Where-Object { 
    $_.IsCustom -eq $false -and (
        $_.Name -like "*admin*" -or 
        $_.Name -in $highPrivilegeRoles
    )
} 

$highPrivRoles | Select-Object Name, Id | 
Export-Csv -Path ".\high-privilege-roles.csv" -NoTypeInformation -Encoding UTF8

# Convert role IDs to policy format
$highPrivRoles | ForEach-Object {
    "/providers/Microsoft.Authorization/roleDefinitions/$($_.Id)"
}
```

#### List Principal IDs for Breakglass Accounts
These commands help identify Object IDs for the `excludedPrincipalIds` parameter:

```powershell
# List emergency access accounts
Get-AzADUser -Filter "startsWith(DisplayName,'emergency')" | 
    Select-Object DisplayName, Id, UserPrincipalName

# List service principals with high privileges
Get-AzADServicePrincipal | Where-Object DisplayName -like "*breakglass*" |
    Select-Object DisplayName, Id, ServicePrincipalType

# Find specific account by display name
$accountName = "Break Glass Account"
Get-AzADUser -DisplayName $accountName | 
    Select-Object DisplayName, Id, UserPrincipalName
```

#### Example Parameter Values
```json
{
    "roleDefinitionIds": {
        "value": [
            "/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635", // Owner
            "/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c"  // Contributor
        ]
    },
    "excludedPrincipalIds": {
        "value": [
            "12345678-1234-1234-1234-123456789012", // Emergency Admin Account
            "87654321-4321-4321-4321-210987654321"  // Break Glass Service Principal
        ]
    },
    "effect": {
        "value": "deny"
    }
}
```