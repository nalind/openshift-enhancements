---
title: allow-users-to-manage-their-own-tokens
authors:
  - "@stlaz"
reviewers:
  - "@sttts"
  - "@deads2k"
  - "@marun"
  - "@vareti"
approvers:
  - "@sttts"
  - "@deads2k"
creation-date: 2020-10-22
last-updated: 2020-10-22
status: implementable
see-also:
- [enhancement: Secure OAuth Resource Storage](https://github.com/openshift/enhancements/pull/323)
replaces:
superseded-by:

---

# Allow Users To Manage Their Own Tokens

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement describes how to allow OpenShift users to be able to list their
access tokens so that they can easily delete an access token specific to an
application, all of their tokens, a token from the past they no longer need etc.

## Motivation

It is quite easy for users to retrieve tokens but it is also not so hard
for them to be able to lose access to them - people don't usually search
their cookies for the value of the token, or don't keep history of the
tokens they retrieve by issuing the `oc login` command multiple times.

### Goals

Allow users to (using either console/CLI):
- list their own tokens
- view details of a specific token
- delete their own tokens

### Non-Goals

- Changing the granularity of tokens to sessions: tokens can be used by many sessions.
  The oauth system is not aware of sessions and won't be with this enhancement.

## Proposal

### User Stories [optional]

#### Story 1
A user is able to list/watch and delete (in both CLI and the web console) all of
the tokens that were issued on their behalf.

#### Story 2
The user can look for specific details of their token like the name of the OAuth2
client that requested issuing the token, token scopes, token expiration date etc.,
making it easier for them to  audit their accesses.

### Implementation Details/Notes/Constraints [optional]

The OAuth API server gets a new API endpoint - `useroauthaccesstokens`. This endpoint
allows to `get`, `list`, `watch` and `delete` tokens that belong to the current user.
The user is determined based on the context of the request, as supplied by the generic
API server logic. This keeps impersonation working, so a user impersonating another
one is capable to perform the above actions on behalf of the impersonated user.

Upon receiving a REST request to the new endpoint, the API server internally
wires the get/list/watch/delete requests to the `oauthaccesstoken` API endpoint.
It then converts the results and returns them as `useroauthaccesstoken` type.
The schema of the `useroauthaccesstoken` type is the same as of `oauthaccesstoken`,
they only differ in kind.

To list/watch the tokens internally, a field selector of the form `userName=<userName>`
is used. It is wrapped in an and-type of a selector around any field selector
user specifies.

When getting or deleting a token, there must be an exact match of `token.userName`
and the `userName` retrieved as described above. If such a match does not occur,
the API server must return "404 - Not Found" so that it is not possible to guess
other people's token object names.

#### Roles modifications

The `basic-user` clusterrole should now additionally receive the following rule:

```yaml
apiGroups:
- oauth.openshift.io
resources:
- useroauthaccesstokens
verbs:
- get
- list
- watch
- delete
```

#### Default table view in CLI

By default, the following fields will be displayed when listing tokens in the
command line:

- metadata.name (`NAME`)
- clientName (`CLIENT NAME`)
- metadata.creationTimestamp (`CREATED`)
- expiresIn (`EXPIRES`)
- redirectURI (`REDIRECT URI`)
- scopes (`SCOPES`)

This matches the fields displayed for the `OAuthAccessToken` resource type.

### Risks and Mitigations

The proposal describes changes to a security sensitive resource that serves
as a means to authenticate to OpenShift.

The level of confidentiality of the `OAuthAccessToken` objects was lowered in
https://github.com/openshift/enhancements/pull/323 and therefore, should a
token leak, it is no longer possible to use the name of such an object to
log in as a different user. On the other hand, the token still contains
information about the user and which services they might be accessing, and
with in the current RBAC rules, it would also allow a malicious user to
remove the token of the victim.

The above risks should be considered during implementation reviews and when
setting up testing for this feature.

## Design Details

### Test Plan

New e2e tests:
1. using the new endpoint, a user can get/list/watch/delete all of the tokens that were issued to them
2. a user cannot get/list/watch/delete tokens for any other user
3. user cannot create, update, patch tokens, neither his own, nor of other users.
4. non-sha256 tokens are not returned by any of the new endpoints.

#### Examples

Without changes to the CLI code, the following will be possible:

```console
$ oc get useroauthaccesstokens
$ oc get -w useroauthaccesstokens
$ oc get useroauthaccesstokens <token-name> [output options]
$ oc delete useroauthaccesstokens <user-owned-token-name>
```

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives (Not Implemented)

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.
