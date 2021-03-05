FullCircle Foundation

Chadwick Kirby

# The Transactional Role-based Access Control (T-RBAC) Data Storage Format

## Abstract

Transactional Role-based Access Control (T-RBAC) is a transaction-log-based data storage format designed to describe the rights of subjects in terms of roles to access resources on the internet. It is organized as a set of JSON schemas. T-RBAC defines the canonical purpose and meaning of the schemas and their components. T-RBAC is well-suited for use on unspent-transaction-output-based blockchains that support transaction metadata large enough to hold descriptive JSON documents.

### Copyright Notice

Copyright (c) 2021 FullCircl Foundation. All rights reserved.

## 1. Introduction

Transactional Role-based Access Control (T-RBAC) is a document format for storing role-based access control (RBAC) policies and their changes over time. It’s implemented as a series of JSON documents, as defined in The JavaScript Object Notation (JSON) Data Interchange Format [RFC 8259].

T-RBAC policies are composed of a series of permissions that apply to resources, granted to subjects, either directly or together as a collection called a “role.”

The current state of the policy can be computed by locating the original policy and all related policy transactions and applying the transactions to the policy in the order that they were added to the data store.

After computing the current state, it is possible to determine what kind of access has been granted to a given subject over a specified resource by locating all permissions assigned directly to the subject and all permissions assigned to the subject through their roles. First all “grant” permissions must be collected, and then “deny” permissions must be subtracted.

## 2. Policies

A T-RBAC policy is a JSON document that conforms to a T-RBAC “Policy” JSON schema. The schema is currently in draft, with the latest version being available here: https://github.com/torus-online/schemas/raw/main/rbac/draft/policy.json

A T-RBAC policy represents the complete state of a policy at any given point in time.

The policy components, stored as JSON object properties, are:
>$schema: the location of the JSON schema to use. Including the schema is important for specifying different versions.

>urn: A universal resource name (URN) that uniquely identifies the policy. It’s recommended to use a UUID to ensure uniqueness. Example: `urn:uuid:f81d4fae-7dec-11d0-a765-00a0c91e6bf6`

>permissionSubjects: An array of permissions along with the subjects that they are assigned to. Permissions are expressed as an object, and subjects must be expressed in URI format. Example:

```json
"permissionSubjects": [{
  "permission": { "mode": "grant", "action": "write", "resource": "urn:uuid:179a9b65-48bb-482e-8cfb-c53d266f85a3" },
  "subjects": [
    "web+cardano://address/addr1qxgnu3h67ctnqfz8hauang4vtmp29nhsp47v56zcqw553lskumdzlg8kqf2sh2ahrvxeqysrndl4spvjngx23y2xuuzs4vpk82"
  ]
}]
```

>roles: An array of objects that define roles as a collection of permissions and member subjects. Example:

```json
"roles": [{
  "name": "User Admin",
    "permissions": [{
    "mode": "grant", "action": "write", "resource": "server/users"
  }],
  "subjects": [
    "web+cardano://address/addr1qxgnu3h67ctnqfz8hauang4vtmp29nhsp47v56zcqw553lskumdzlg8kqf2sh2ahrvxeqysrndl4spvjngx23y2xuuzs4vpk82"
  ]
}]
```
## 3. Policy Transactions

A T-RBAC policy transaction describes a transformation that should be applied to a policy as a single transaction to update its state. The schema is currently in draft, with the latest version being available here: https://github.com/torus-online/schemas/raw/main/rbac/draft/policy-tx.json

A T-RBAC policy transaction applies a transaction method to a policy, as specified by its unique URN. A transaction document includes the following properties:

>$schema: the location of the JSON schema to use. Including the schema is important for specifying different versions.

>policyUrn: The unique URN of the policy to apply the transaction to

>method: The type of transaction to apply to the policy. Valid values are: "patch", "put", and "delete".

>body: The body of the transaction. Only used in conjunction with the “patch” and “put” methods.

## 3.1. Transaction Methods

## 3.2. Patch

The patch method specifies that the contents of the body property will be used to describe a transaction where the policy is partially updated. When the patch method is specified, body must conform to the JSON-Patch schema, as defined here: http://json.schemastore.org/json-patch.

Only JSON-Patch operations that produce a schema-compliant T-RBAC policy document are valid. When calculating policy state, any patch transaction that produced a policy which is not compliant with the policy schema shall be ignored as not valid.

## 3.3. Put

The put method specifies that the body property will be an entire policy document that will completely replace the policy. When the put method is specified, body must conform to the T-RBAC policy schema, as defined here: https://github.com/torus-online/schemas/raw/main/rbac/draft/policy.json

## 3.4. Delete

The delete method indicates that the policy is no longer valid and has been destroyed. The body property should be excluded and will be ignored if included.

## 4. Ownership

Ownership of the policy is defined within the policy itself. Subjects must be granted write access to the policy itself, as specified in a permission object by its URN, in order to be authorized to post policy transactions.

Any transactions that are posted by unauthorized subjects will be ignored when computing the policy state.

## 5. Computing State

In order to compute the state of a policy as of a transaction, all of its prior, related transactions must be collected in the order that they were created, all the way back to the initial creation of the policy document. Starting with the initial policy, all transactions must be applied to that policy in order.

Transactions that do not meet the criteria for validity will be ignored.

The criteria for validity are:
1. The transaction must be posted by a user that was authorized inside the policy as of the state of the last transaction in the log.
1. The transaction must be entirely compliant with the policy transaction schema
1. The resulting policy state must be entirely compliant with the policy schema

## 6.  Examples

This is a T-RBAC policy:
```json
{
   "$schema": "https://github.com/torus-online/schemas/raw/main/rbac/draft/policy.json",
   "urn": "urn:uuid:179a9b65-48bb-482e-8cfb-c53d266f85a3",
   "permissionSubjects": [{
       "permission": { "mode": "grant", "action": "write", "resource": "urn:uuid:179a9b65-48bb-482e-8cfb-c53d266f85a3" },
       "subjects": [ "web+cardano://address/addr1qxgnu3h67ctnqfz8hauang4vtmp29nhsp47v56zcqw553lskumdzlg8kqf2sh2ahrvxeqysrndl4spvjngx23y2xuuzs4vpk82" ]
   }],
   "roles": [
       {
           "name": "User Admin",
           "permissions": [{
               "mode": "grant", "action": "write", "resource": "server/users"
           }],
           "subjects": [ "web+cardano://address/addr1qxgnu3h67ctnqfz8hauang4vtmp29nhsp47v56zcqw553lskumdzlg8kqf2sh2ahrvxeqysrndl4spvjngx23y2xuuzs4vpk82" ]
       }
   ]
}
```

It grants the following subject with sole rights to update the policy in further transactions: `web+cardano://address/addr1qxgnu3h67ctnqfz8h...uuzs4vpk82`

It also defines the User Admin role as having write access to the “server/users” resource, and gives the following subject membership to that role: `web+cardano://address/addr1qxgnu3h67ctnqfz8h...uuzs4vpk82`

This is a T-RBAC Policy Transaction:

```json
{
   "$schema": "https://github.com/torus-online/schemas/raw/main/rbac/draft/policy-tx.json",
   "policyUrn": "urn:uuid:179a9b65-48bb-482e-8cfb-c53d266f85a3",
   "method": "patch",
   "body": { "op": "add", "path": "/permissionSubjects/0/subjects/-", "value": "web+cardano://address/addr1qxgnu3h67ctnqfz8hauang4vtmp29nhsp47v56zcqw553lskumdzlg8kqf2sh2ahrvxeqysrndl4spvjngx23y2xuuzs4vpk82" }
}
```

It specifies that the following subject will be assigned to the first defined permission: `web+cardano://address/addr1qxgnu3h67ctnqfz8h...uuzs4vpk82`

```json
{
   "$schema": "https://github.com/torus-online/schemas/raw/main/rbac/draft/policy-tx.json",
   "policyUrn": "urn:uuid:179a9b65-48bb-482e-8cfb-c53d266f85a3",
   "method": "put",
   "body": {
       "urn": "urn:uuid:179a9b65-48bb-482e-8cfb-c53d266f85a3",
       "permissionSubjects": [{
           "permission": { "mode": "grant", "action": "write", "object": "urn:uuid:179a9b65-48bb-482e-8cfb-c53d266f85a3" },
           "subjects": [ "web+cardano://address/addr1qxgnu3h67ctnqfz8hauang4vtmp29nhsp47v56zcqw553lskumdzlg8kqf2sh2ahrvxeqysrndl4spvjngx23y2xuuzs4vpk82" ]
       }],
       "roles": [
           {
               "name": "User Admin",
               "permissions": [{
                   "mode": "grant", "action": "write", "object": "server/users"
               }],
               "subjects": [ "web+cardano://address/addr1qxgnu3h67ctnqfz8hauang4vtmp29nhsp47v56zcqw553lskumdzlg8kqf2sh2ahrvxeqysrndl4spvjngx23y2xuuzs4vpk82" ]
           }
       ]
   }
}
```

This transaction specifies that the entire policy will be replaced with a new one, as defined in the body property.

```json
{
   "$schema": "https://github.com/torus-online/schemas/raw/main/rbac/draft/policy-tx.json",
   "policyUrn": "urn:uuid:179a9b65-48bb-482e-8cfb-c53d266f85a3",
   "method": "delete"
},
```

This transaction deletes the policy with URN: `urn:uuid:179a9b65-48bb-482e-8cfb-c53d266f85a3`

## 7.  References

[RFC 8259] Bray, T., “The JavaScript Object Notation (JSON) Data Interchange Format”
https://www.rfc-editor.org/info/rfc8259