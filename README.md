# SCIM

This repo defines the proposed [SCIM](https://tools.ietf.org/html/rfc7643) interface to be provided by IGA
implementations in the Norwegian higher education sector. The main use case for this API is to provide
for IntArk-style provisioning of user accounts.

A functional mockup of this API is available from
[UiBs API Gateway](https://api-uib.intark.uh-it.no/#!/apis/91a73d99-d9b2-452a-a73d-99d9b2e52a9a/detail).

The standard paths of SCIM are `/Users`, `/Users/{id}`, `/Groups` and `/Groups/{id}`. We only care for
the user endpoints for now.

## Minimal implementation requirements

This section defines what the lazy implementer might get away with.

* Implement `/Users` with ability to page throgh the available accounts. An implementation might choose to just expose the Feide-enabled accounts.
* Implement `/Users?filter=userName eq "gaa041@uib.no"` to make it possible to look up a specific account
* Implement `/Users/{id}` fetch data for the specified account
* Make `/Groups` functional, but it's fine for it to just return the empty `ListResponse`.
* Post MQ message when a user object is created, modified or deleted.

The following fields should be provided on user objects.

* `.id`
* `.meta`
* `.username`
* `.active`
* `.name.formatted`
* `.name.givenName`
* `.name.familyName`
* `.emails[].type == "work"` with a corresponding `.value`
* `.enterprise.employeeNumber`
* `.no:edu:scim:user.eduPersonPrincipalName`

## Events

Updates to the objects exposed in this API is signaled by events to the IntArk MQ
and follows the proposed
[SCIM Event Extension](https://tools.ietf.org/html/draft-hunt-scim-notify-00) structure.
Since IntArk prefers shallow messages, we don't include the `.values` attribute.

These events are encoded in JSON and looks like this:

```
{
  "schemas: ["urn:ietf:params:scim:schemas:notify:2.0:Event"],
  "resourceUris": [
     "https://gw-uib.intark.uh-it.no/iga/scim/v2/Users/362ff2749bfb11eabbd5600308a4105a"
  ],
  "type":"MODIFY",
  "attributes": ["emails", "no:edu:user"],
}
```

## Recommended features

This section expands on the minimal requirements and define some features that
might be useful and that we prefer all implementations to consider.

* Implement functional `/Groups` and `/Groups/{id}` that expose the same groups available from LDAP/AD.
* More attributes on user objects, especially `.phoneNumbers` and `.enterprise.manager`.
* Search for users by name and other attributes.
* Implement `/Persons` and `/Persons/{id}` endpoints. A schema for person objects is yet to be defined.


## Field name specification

### User `.id`

This should preferably be UUID-style string, but the service is free to
use other formats, like letting the `.username` be the `.id`.  It should be possible
to request this user with an URL-path of `/Users/{id}` where the `{id}` is replaced
by the value of this field.

### User `.username`

The format of the username value should be `{local-username}@{fqdn}`.

Example value: `gaa041@uib.no`

### User `.active`

Boolean value which is set to `false` for accounts that should be disabled.
Users should not be able to login using this account. Any active session
using this account should also be terminated.

### User `.externalId`

The identifier of the person that owns this account. It is preferable that
`/Persons/{externalId}` fetches information on the person.  For UH-IAM this
will be the UH-ID (another UUID value).

### User `.emails`

The email addresses associated with this account. The email addresses are
tagged with a type field.  The tag "work" is used for the main email address,
even for students.

Example value:
```
[
    {
        "type": "work",
        "value": "Gisle.Aas@uib.no"
    },
    {
        "type": "internal",
        "value": "gaa041@uib.no"
    },
]
```

### User `.enterprise`

This is the standard SCIM enterprise extension object.
The real full name of this attribute is `urn:ietf:params:scim:schemas:extension:enterprise:2.0:User`
but we shortened it in this description to `.enterprise`.

### User `.enterprise.employeeNumber`

This is the DFØ ID for the employee that owns this account.
This field isn't present for student-only accounts.

### User `.no:edu:scim:user`

This attibute contains the object where we extend the user object with
fields specific to Norwegian UH domain. The string is also the name
of a schema and should be found in the `.schemas` attribute as well.

### User `.no:edu:scim:user.eduPersonPrincipalName`

The Feide-ID of this user account is specified in this field.
This field is absent for accounts not available through Feide.

It will have the same value as the `eduPersonPrincipalName` attribute
in LDAP and specified [norEdu*](https://docs.feide.no/reference/schema/attributes/edupersonprincipalname.html#saml-attribute-edupersonprincipalname).

Example value: `gaa041@uib.no`

### User `.no:edu:scim:user.userPrincipalName`

This is the login name for Microsoft login.
This is the same attribute as `userPrincipalName` in AD and Azure AD.

It would be a good idea for this value to be the same as `.eduPersonPrincipalName`
but UiB has unfortunately diverged.

Example value: `Gisle.Aas@uib.no`