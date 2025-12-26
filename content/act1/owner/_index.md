+++
date = '2025-12-23T15:59:59+01:00'
draft = false
hidden = false
title = 'Owner'
weight = 12
+++

![James Goose](/images/act1/goose-steampunk.png)

## Objective

| Difficulty | Description |
| ---------- | ----------- |
| 1/5 | Help Goose James near the park discover the accidentally leaked SAS token in a public JavaScript file and determine what Azure Storage resource it exposes and what permissions it grants. |

## Goose James mission statement

> CLUCK CLU... I think I might be losing my mind. All the elves are gone and I'm still hearing voices.
> 
> ----
> The Neighborhood HOA uses Azure for their IT infrastructure.
> 
> The Neighborhood network admins use RBAC fo access control.
> 
> Your task is to audit their RBAC configuration to ensure they're following security best practices.
> 
> They claim all elevated access uses PIM, but you need to verify there are no permanently assigned Owner roles.

## Terminal

This objective focuses on reviewing identity and access management in a cloud environment to identify overly permissive role assignments. By examining Azure RBAC and the use of Privileged Identity Management (PIM), the task highlights the risk of permanently assigned Owner roles and reinforces the importance of least privilege and time-bound administrative access.

![Objective briefing](/images/act1/act1-owner-1.png)

### Query parameter

Prototype command:

```bash
az account list --query "[].name"
```

![List accounts](/images/act1/act1-owner-2.png)

### Conditional filtering

Prototype command:

```bash
az account list --query "[?state=='Enabled'].{Name:name, ID:id}"
```

![List accounts](/images/act1/act1-owner-3.png)

### Looking at owner of first listed subscription

Prototype command:

```bash
az role assignment list --scope "/subscriptions/{ID of first Subscription}" --query [?roleDefinition=='Owner']
```

Updated command:

```bash
az role assignment list --scope "/subscriptions/2b0942f3-9bca-484b-a508-abdae2db5e64" --query [?roleDefinition=='Owner']
```

![Owner of the first subscription](/images/act1/act1-owner-4.png)

### Looking at other owners listed subscriptions

Let's run the previous command against the other subscriptions to see what we come up with:

```bash
az role assignment list --scope "/subscriptions/4d9dbf2a-90b4-4d40-a97f-dc51f3c3d46e" --query [?roleDefinition=='Owner']
az role assignment list --scope "/subscriptions/065cc24a-077e-40b9-b666-2f4dd9f3a617" --query [?roleDefinition=='Owner']
az role assignment list --scope "/subscriptions/681c0111-ca84-47b2-808d-d8be2325b380" --query [?roleDefinition=='Owner']
```

Subscription `065cc24a-077e-40b9-b666-2f4dd9f3a617` yielded something interesting: 

```json
[
  {
    "condition": "null",
    "conditionVersion": "null",
    "createdBy": "85b095fa-a9b4-4bdc-a3af-c9f95ebb8dd6",
    "createdOn": "2025-09-10T15:45:12.439266+00:00",
    "delegatedManagedIdentityResourceId": "null",
    "description": "null",
    "id": "/subscriptions/065cc24a-077e-40b9-b666-2f4dd9f3a617/providers/Microsoft.Authorization/roleAssignments/b1c69caa-a4d6-449a-a090-efacb23b55f3",
    "name": "b1c69caa-a4d6-449a-a090-efacb23b55f3",
    "principalId": "2b5c7aed-2728-4e63-b657-98f759cc0936",
    "principalName": "PIM-Owners",
    "principalType": "Group",
    "roleDefinitionId": "/subscriptions/065cc24a-077e-40b9-b666-2f4dd9f3a617/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635",
    "roleDefinitionName": "Owner",
    "scope": "/subscriptions/065cc24a-077e-40b9-b666-2f4dd9f3a617",
    "type": "Microsoft.Authorization/roleAssignments",
    "updatedBy": "85b095fa-a9b4-4bdc-a3af-c9f95ebb8dd6",
    "updatedOn": "2025-09-10T15:45:12.439266+00:00"
  },
  {
    "condition": "null",
    "conditionVersion": "null",
    "createdBy": "85b095fa-a9b4-4bdc-a3af-c9f95ebb8dd6",
    "createdOn": "2025-09-10T16:58:16.317381+00:00",
    "delegatedManagedIdentityResourceId": "null",
    "description": "null",
    "id": "/subscriptions/065cc24a-077e-40b9-b666-2f4dd9f3a617/providers/Microsoft.Authorization/roleAssignments/6b452f58-6872-4064-ae9b-78742e8d987e",
    "name": "6b452f58-6872-4064-ae9b-78742e8d987e",
    "principalId": "6b982f2f-78a0-44a8-b915-79240b2b4796",
    "principalName": "IT Admins",
    "principalType": "Group",
    "roleDefinitionId": "/subscriptions/065cc24a-077e-40b9-b666-2f4dd9f3a617/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635",
    "roleDefinitionName": "Owner",
     "roleDefinitionName": "Owner",
    "scope": "/subscriptions/065cc24a-077e-40b9-b666-2f4dd9f3a617",
    "type": "Microsoft.Authorization/roleAssignments",
    "updatedBy": "85b095fa-a9b4-4bdc-a3af-c9f95ebb8dd6",
    "updatedOn": "2025-09-10T16:58:16.317381+00:00"
  }
]
```

What is interesting here? 

1. **Group `PIM-Owners` (`principalId: 2b5c7aed-2728-4e63-b657-98f759cc0936`) has permanent `Owner`** on the full subscription — weak PIM/JIT hygiene.
2. **Group `IT Admins` (`principalId: 6b982f2f-78a0-44a8-b915-79240b2b4796`) is also assigned `Owner`** — likely over-privileged for a broad admin group.
3. **Both assignments are unrestricted** — no conditions, no constraints, full subscription-level control.
4. **Both assignments were created by the same identity (`createdBy: 85b095fa-a9b4-4bdc-a3af-c9f95ebb8dd6`)**

### Figuring out membership of group

Prototype command: 

```bash
az ad member list
```

Refined command aiming directly at "IT Admins" as that looked most promising: 

```bash
az ad member list --group 6b982f2f-78a0-44a8-b915-79240b2b4796 | less
```

![Group information 1](/images/act1/act1-owner-5.png)

Turns out we found a nested group

### Nested groups

Rerunning the last command, but with a different id: 

```bash
az ad member list --group 631ebd3f-39f9-4492-a780-aef2aec8c94e | less
```

![Group information 2](/images/act1/act1-owner-6.png)

Full output provided below: 

```json
[
  {
    "@odata.type": "#microsoft.graph.user",
    "businessPhones": [
      "+1-555-0199"
    ],
    "displayName": "Firewall Frank",
    "givenName": "Frank",
    "id": "b8613dd2-5e33-4d77-91fb-b4f2338c19c9",
    "jobTitle": "HOA IT Administrator",
    "mail": "frank.firewall@theneighborhood.invalid",
    "mobilePhone": "+1-555-0198",
    "officeLocation": "HOA Community Center - IT Office",
    "preferredLanguage": "en-US",
    "surname": "Firewall",
    "userPrincipalName": "frank.firewall@theneighborhood.onmicrosoft.com"
  }
]
```

## Goose James closing words

After solving, James says:

> You found the permanent assignments! CLUCK! See, I'm not crazy - the security really WAS misconfigured. Now maybe I can finally get some peace and quiet...
