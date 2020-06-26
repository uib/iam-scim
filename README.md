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

User object fields:

* `.id`
* `.username`
* `.name.formatted`
* `.name.givenName`
* `.name.familyName`
* `.emails[].type == "work"` with a corresponding `.value`
* `.enterprise.employeeNumber`
* `.no:edu:person.eduPersonPrincipalName`