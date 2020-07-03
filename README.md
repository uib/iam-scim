# SCIM

This repo defines the proposed [SCIM](https://tools.ietf.org/html/rfc7643) interface to be provided by IGA
implementations in the Norwegian higher education sector. The main use case for this API is to provide
for IntArk-style provisioning of user accounts.

A functional mock-up of this API is available from
[UiBs API Gateway](https://api-uib.intark.uh-it.no/#!/apis/91a73d99-d9b2-452a-a73d-99d9b2e52a9a/detail).

The standard paths of SCIM are `/Users`, `/Users/{id}`, `/Groups` and `/Groups/{id}`. We only care for
the user endpoints for now.

## Persons vs Accounts

SCIM is basically designed as a REST-style replacement for LDAP.  As such it
has the same ambiguity on what the User objects represents — are they people or
are they accounts that belong to people (and other entities).

Here we declare them to be _accounts_ and we suggest that we might extend our
SCIM implementation with _Person_ objects later. In this model a person might be
the owner of multiple accounts.  We also consider one of these accounts as this
person's primary account.  The User objects of the primary account will have a
mix of attributes describing the account and attributes describing the person.

We might also have accounts that doesn't belong to any person. This might
be accounts representing devices, applications or other systems.


## Minimal implementation requirements

This section defines what the lazy implementer might get away with.

* Implement `/Users` with ability to page through the available accounts. An implementation might choose to just expose the Feide-enabled accounts.
* Implement `/Users?filter=userName eq "..."` to make it possible to look up a specific account
* Implement `/Users/{id}` fetch data for the specified account
* Make `/Groups` functional, but it's fine for it to just return the empty `ListResponse`.
* Post MQ message when a user object is created, modified or deleted.

The following fields should minimally be provided on user objects.

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

Example topic for the message is "no.uib.iga.scim.user.modify".

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
to request the JSON object representing this user with an URL-path of `/Users/{id}`
where the `{id}` is replaced by the value of this field.

### User `.meta`

Standard SCIM meta information on this object.

### User `.username`

The format of the username value should be `{local-username}@{fqdn}`.  SCIM allows
bare usernames, as well as usernames qualified with a domain. For consistency we always
return fully qualified names.

Example value: `gaa041@uib.no`

### User `.displayName`

For the primary account this is the same as `.name.formatted`.
For other accounts this can be any string that is suitable for
explaining the purpose of this account.

### User `.name`

This is the name of the owner of this account.  This attribute is
mandatory for primary accounts.  Example value:

```
{
    "formatted": "Gisle Aas",
    "familyName": "Aas",
    "givenName": "Gisle"
}
```

### User `.active`

Boolean value which is set to `false` for accounts that should be disabled.
Users should not be able to login using this account. Any active session
using this account should also be terminated.

### User `.externalId`

The identifier of the person that owns this account. It is preferable that
`/Persons/{externalId}` fetches information on the person.  For UH-IAM this
field will be the UH-ID (a person oriented UUID value).

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

### User `.phoneNumbers`

The public phone numbers associated with this account.  The phone numbers are
tagged with a type filed. The tag "work" used for prefered contact option.  The
tag "mobile" can be used when the mobile phone is not the prefered option.  The
phone numbers themselves are represented in the full international format
prefixed with "+" and without internal spaces or dashes.

Example value:

```
[
    { "value": "+4793241450", "type": "work"   },
    { "value": "+4793241450", "type": "mobile" }
]
```

### User `.profileUrl`

URL of the university home page for the owner of this account.

### User `.entitlements`

List of strings representing entitlements for this account. Not sure if we
should use this for anything.

### User `.roles`

List of business roles associated with this user.
Apply the tags "staff", "guest" or "student" when approripate. This
list of roles might be extended later.
Universites can introduce private roles by prefixing them with
their reverse domain name, e.g. "no.uib.breiflabb".

### User `.enterprise`

This is the standard SCIM enterprise extension object.
The real full name of this attribute is `urn:ietf:params:scim:schemas:extension:enterprise:2.0:User`
but we shortened it in this description to `.enterprise`.

### User `.enterprise.employeeNumber`

This is the DFØ ID for the employee that owns this account.
This field isn't present for student-only accounts.

### User `.enterprise.costCenter`

This is the account number used for expenses related to the owner of this account.
Not sure if this is useful.

### User `.enterprise.organization

The name of the organisation that the owner of this account belongs to.
For staff and students this will be the name of the school.  For guest this
might be the name of the organisation they come from.

### User `.enterprise.division`

The name of primary faculty that the owner of this account belongs to.

### User `.enterprise.departement`

The name of primary intitute that the owner of this account belongs to.

### User `.enterprise.manager`

For primary accounts, this is set to reference the primary user account of the manager of the person that owns this account.
All non-primary accounts should set this to reference the primary account of their owner.

Additional information on the manager can be obtained from `/Users/{enterprise.manager.id}`.
The `.enterprise.manager.displayName` is just a copy of the `.displayName` of the manager account itself.

Example value:
```
{
    "id": "452ff2749bfb11eabbd5600308a4105a",
    "displayName": "Nina Kaurel"
}
```

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
