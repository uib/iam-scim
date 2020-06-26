This API exposes users at UiB using the standard [SCIM](https://tools.ietf.org/html/rfc7643) format. Currently this only contains test data while we explore how to define and extend this API. The plan is that the new IGA we are piloting fall 2020 will provide this interface.

The endpoints that are currently implemeneted are:

* `GET https://gw-uib.intark.uh-it.no/iga/scim/v1/Users`
* `GET https://gw-uib.intark.uh-it.no/iga/scim/v1/Users/{id}`

Suggested user attribute mappings:

* `.id` is the account id (UUID)
* `.username` is the eduPersonPrinicalName (also called Feide-ID)
* `.externalId` is the UH-ID
* `.emails[].type == "work"` is the main email address both for employees and students
* `enterprise.employeeNumber` is the DFØ-id
* `enterprise.costCenter` is the matches the `.kostnadssed` field in the DFØ ansatte object. It's not filed in for students.
* `enterprise.manager` is the manager of the org unit of the main affilitation. It's not filled in for students.

## Events

Updates to the objects exposed in this API is signaled by evenst to the IntArk MQ and follows the proposed "[SCIM Event Extension](https://tools.ietf.org/html/draft-hunt-scim-notify-00) structure. These events look like this:

```
{
  "schemas: ["urn:ietf:params:scim:schemas:notify:2.0:Event"],
  "resourceUris": [
     "https://gw-uib.intark.uh-it.no/iga/scim/v1/Users/362ff2749bfb11eabbd5600308a4105a"
  ],
  "type":"MODIFY",
  "attributes": ["emails", "enterprise.manager"],
}
```
