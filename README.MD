# Deny assignments of high-privilege roles (Parametrized) with breakglass exception (PIM-aware)

This Azure Policy definition denies creation or scheduling of role assignments and PIM-related role eligibility/assignment operations for a configurable list of high-privilege roleDefinitionIds, except for explicitly excluded "breakglass" principals. It is PIM-aware and evaluates both direct role assignments and Privileged Identity Management (PIM) schedule/eligibility resources so that scheduled/elevated requests are also covered.

Policy file
- deny-high-privilege-roles-parametrized-with-breakglass-exception-pim-aware.json  
  (source/permalink: https://github.com/abrarhuss/AzPrivilegedUserPolicies/blob/f23fb69ad6a5d153a5b95762c4ce447e99a6fd37/deny-high-privilege-roles-parametrized-with-breakglass-exception-pim-aware.json)

Summary
- Prevents assignment or scheduling of specified high-privilege roleDefinitionIds (for example, Owner, User Access Administrator).
- Covers resource types:
  - Microsoft.Authorization/roleAssignments
  - Microsoft.Authorization/roleEligibilitySchedules
  - Microsoft.Authorization/roleAssignmentSchedules
  - Microsoft.Authorization/roleEligibilityScheduleRequests
  - Microsoft.Authorization/roleAssignmentScheduleRequests
- Allows a list of excluded principal object IDs (breakglass accounts) to be exempt from the deny.
- Parameterized so you can supply roleDefinitionIds and excludedPrincipalIds at assignment time or via parameters file.
- Default effect is "deny" (you may set to "audit" first to evaluate impact).

Parameters
- roleDefinitionIds (Array) — Full resource IDs of role definitions to block. Example:
  - /providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635 (Owner)
- excludedPrincipalIds (Array) — Azure AD object IDs (users or service principals) to exempt (breakglass).
- effect (String) — One of: "deny", "audit", "disabled". Default: "deny".

Why use this policy
- Enforces principle of least privilege by preventing uncontrolled assignment of highly privileged roles.
- Blocks both direct role assignments and PIM related scheduling/eligibility operations to reduce privilege escalation via PIM workflows.
- Provides an emergency breakglass mechanism (explicitly excluded principals) for controlled exceptions.

Recommended deployment workflow
1. Test in audit mode
   - Create the policy definition with effect set to "audit" or supply effect parameter "audit" when assigning to evaluate impact and find existing violations before enforcing deny.
2. Tune parameters
   - Populate roleDefinitionIds with the high-privilege role IDs you want to control.
   - Add known, controlled breakglass principal object IDs to excludedPrincipalIds.
3. Assign at the appropriate scope
   - Assign the policy at subscription, management group, or tenant scope depending on your governance model.
4. Promote to deny
   - After validating the audit results and adjusting exceptions, change effect to "deny" to enforce.

Deployment examples

Azure CLI (create definition and assign)
- Create policy definition:
  az policy definition create \
    --name "Deny-High-Privilege-Roles-PIM-Aware" \
    --display-name "Deny assignments of high-privilege roles (Parametrized) with breakglass exception (PIM-aware)" \
    --description "Deny direct and PIM-related role assignments for specified roles unless principal is excluded (breakglass)." \
    --rules ./deny-high-privilege-roles-parametrized-with-breakglass-exception-pim-aware.json \
    --mode All

- Assign policy with parameter values (example)
  az policy assignment create \
    --name "DenyHighPrivRoles-Assignment" \
    --scope /subscriptions/<subscriptionId> \
    --policy "Deny-High-Privilege-Roles-PIM-Aware" \
    --params '{
      "roleDefinitionIds": {"value": ["/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635"]},
      "excludedPrincipalIds": {"value": ["<breakglass-objectId-1>", "<breakglass-objectId-2>"]},
      "effect": {"value": "deny"}
    }'

PowerShell (example)
- Create policy definition:
  New-AzPolicyDefinition -Name "Deny-High-Privilege-Roles-PIM-Aware" `
    -DisplayName "Deny assignments of high-privilege roles (Parametrized) with breakglass exception (PIM-aware)" `
    -Policy ./deny-high-privilege-roles-parametrized-with-breakglass-exception-pim-aware.json

- Assign with parameters (example):
  $params = @{
    "roleDefinitionIds" = @{ "value" = @("/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635") }
    "excludedPrincipalIds" = @{ "value" = @("<breakglass-objectId>") }
    "effect" = @{ "value" = "deny" }
  }
  New-AzPolicyAssignment -Name "DenyHighPrivRoles-Assignment" -Scope "/subscriptions/<subscriptionId>" -PolicyDefinition (Get-AzPolicyDefinition -Name "Deny-High-Privilege-Roles-PIM-Aware") -PolicyParameterObject $params

Notes & considerations
- Correct roleDefinitionId format is required: full resource id under /providers/Microsoft.Authorization/roleDefinitions/{guid}.
- The policy evaluates resource types used by Azure RBAC and PIM workflows; it will block both direct assignments and PIM-created schedules/requests that match the roleDefinitionIds and are not for excluded principals.
- Use "audit" to observe existing assignments/requests before enforcing "deny".
- Breakglass principals are a safety valve. Keep them tightly controlled, rotated, and monitored.
- If you need to allow an entire managed identity or workspace to keep assigning roles, add that principal's objectId to excludedPrincipalIds. Be deliberate with exclusions — they bypass the policy.
- This policy is resource-type driven. It will not retroactively remove existing assignments — it will only prevent creation/scheduling while in effect. Use remediation scripts if you need to remove existing privileged assignments.

Testing
- Set effect to "audit" and attempt to create a role assignment for a blocked roleDefinitionId using a non-excluded principal. Verify the policy violation is logged in Policy -> Assignments -> Compliance.
- Attempt the same action using a principal added to excludedPrincipalIds and verify it is allowed.
- Test PIM operations by creating an eligibility or assignment schedule/request for the blocked role and check the policy behavior.

Limitations
- The policy requires correct, explicit roleDefinition resource IDs. Human-friendly names are not matched.
- If Azure introduces new PIM-related resource types in the future, those resource types will need to be added if they are not covered.
- The policy prevents creation/scheduling but does not automatically remediate or remove pre-existing assignments.

Recommended monitoring & alerting
- Monitor policy compliance in Azure Policy and set up alerts or Logic Apps to notify security or platform teams on new "audit" or "deny" events.
- Integrate with Azure AD sign-in and audit logs to monitor use of breakglass principals.

Change log
- 1.0.0 — Initial policy creation (parametrized, PIM-aware, breakglass support).

License & attribution
- MIT-style use or per repository license. See repository root for license details.

Author / Contact
- Repository: abrarhuss/AzPrivilegedUserPolicies  
- Policy file source: deny-high-privilege-roles-parametrized-with-breakglass-exception-pim-aware.json