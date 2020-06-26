# SCIM

This repo defines the proposed [SCIM](https://tools.ietf.org/html/rfc7643) interface to be provided by IGA
implementations in the Norwegian higher education sector. The main use case for this API is to provide
for IntArk-style provisioning of user accounts.

A functional mockup of this API is available from
[UiBs API Gateway](https://api-uib.intark.uh-it.no/#!/apis/91a73d99-d9b2-452a-a73d-99d9b2e52a9a/detail).

The standard paths of SCIM are `/Users`, `/Users/{id}`, `/Groups` and `/Groups/{id}`. We only care for
the user endpoints for now.

## Minimal implementation requirements

* Implement `/Users` with ability to page throgh the available accounts
* Implement `/Users?filter=userName eq "gaa041@uib.no"` to make it possible to look up a specific account
* Implement `/Users/{id}` fetch data for the specified account
* Post MQ message when a user object is created, modified or deleted.

The following fields should be provided on user objects.

* `.id`
* `.username`
* `.active`
* `.name.formatted`
* `.name.givenName`
* `.name.familyName`
* `.emails[].type == "work"` with a corresponding `.value`
* `.enterprise.employeeNumber`
* `.no:edu:scim:user.eduPersonPrincipalName`

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