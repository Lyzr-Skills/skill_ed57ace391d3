# identity-service SCIM and role assignment: questions

2026-09-18

We are integrating SailPoint, our IAM platform, with identity-service over SCIM 2.0.
The following came up while configuring that connector and while testing against our
prod deployment. Where we cite behavior, we observed it directly against the running
service.

For each item it would help to know which have answers today and which need
investigation, plus rough timelines for anything that is planned.

---

## 1. Two SCIM discovery endpoints return 404

`GET /scim/v2/ResourceTypes/User` and
`GET /scim/v2/Schemas/urn:ietf:params:scim:schemas:core:2.0:User` both return 404.

The collection endpoints do work, and they advertise exactly those ids:
`/ResourceTypes` returns an entry with `id: "User"`, and `/Schemas` returns
`urn:ietf:params:scim:schemas:core:2.0:User`. RFC 7644 section 4 expects individual
resources to be retrievable by id from both endpoints.

SailPoint's connector prerequisites list all five discovery endpoints as required.

Are the two per-id endpoints implemented anywhere, or planned? If not, we will
configure SailPoint to skip the requirement, but we would like to know which it is.

## 2. GET /Groups does not return members

`GET /scim/v2/Groups` returns groups without their `members`. A direct
`GET /scim/v2/Groups/{id}` on the same group returns the full member list.

Is that intended? We ask because a provisioning connector that reconciles membership
from the list response will see an empty group, and depending on its logic may treat
the existing members as removals.

## 3. ETag support

`GET /scim/v2/ServiceProviderConfig` reports `etag.supported` as false.

Is ETag support planned? Without it, a connector has no cheap way to detect changes
and does full comparisons on every aggregation.

## 4. The 500-member reconciliation cap

Your SCIM documentation states that groups larger than 500 members are not fully
reconciled on a full-replace `PUT`, asks customers to report it, and says the cap is
being lifted.

What is the status of that work? We are below the cap today but want to know before
we grow into it.

## 5. Filter support

Filtering appears to be equality only: `userName eq` on Users, and `displayName eq`
or `externalId eq` on Groups. Anything else returns `400 invalidFilter`.

Is that the intended permanent surface, or is broader filter support planned? Either
answer is fine, we just need to tell our IAM team what to configure.

## 6. A user's role is set at first login and never re-evaluated

This is the most important item on the list for us.

We observe that a user's parent-organization role is written when they first join the
organization and is not re-applied on subsequent logins, because
`add_user_to_organization` is called without `resync_role` in
`pagos/app/organizations/service.py`.

The consequence is that changing a user's group membership after their first login has
no effect on their role. Automated promotions and demotions driven from our identity
provider therefore never reach existing users, and have to be corrected by hand in the
Studio UI.

Is this intended? Is there a supported way to have a role re-resolved from group
membership on each login, or on demand? This determines whether automated entitlement
management is viable for us at all.

## 7. Swagger or OpenAPI specification for the SCIM endpoints

Our IAM team has asked for this directly, so they can configure and validate the
connector. Specifically they want the request and response body shapes, and the
request headers, for the `/scim/v2/*` endpoints.

What we have determined ourselves: the auth header is `Authorization: Bearer <token>`,
the content type is `application/scim+json`, and we know the shape of stored User and
Group records from reading them back.

What we cannot find documented anywhere: the exact request bodies accepted on `POST`,
`PUT` and `PATCH` for Users and Groups. We have been working these out empirically,
which means we trial things like group creation in a non-production environment first
rather than working from a specification.

Does an OpenAPI or Swagger document already exist for `/scim/v2/*`? If not, could one
be produced? Failing that, a written reference of request and response bodies per
endpoint would cover what our IAM team needs.
