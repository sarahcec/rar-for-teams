# OAuth 2.0 Rich Authorization Requests Profile: Team Access Tokens

## Abstract

This document defines a profile of OAuth 2.0 Rich Authorization Requests (RAR) [RFC9396] for requesting access tokens that grant permissions to a workload acting on behalf of a team, rather than an individual user. It introduces a new authorization details type that allows clients to specify a team identifier, its members (using "sub_ids" as an array of subject identifiers, consistent with Security Event Token profiles such as RISC events), and an operand ("AND" or "OR") to determine how team member permissions are combined (intersection or union). The resulting access token uses a workload identifier in the `sub` claim, and its permissions are restricted to the intersection of the workload’s permissions and the team’s aggregated permissions, ensuring the workload cannot use team access tokens to accomplish privilege escalation.

## Status of This Memo

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79.

Internet-Drafts are working documents of the Internet Engineering Task Force (IETF). Note that other groups may also distribute working documents as Internet-Drafts. The list of current Internet-Drafts is at https://datatracker.ietf.org/drafts/current/.

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time. It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

## Copyright Notice

Copyright (c) 2025 IETF Trust and the persons identified as the document authors. All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (https://trustee.ietf.org/license-info) in effect on the date of publication of this document. Please review these documents carefully, as they describe your rights and restrictions with respect to this document. Code Components extracted from this document must include Simplified BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Simplified BSD License.

## Table of Contents

- [1. Introduction](#1-introduction)
- [2. Terminology](#2-terminology)
- [3. Team Access Authorization Details Type](#3-team-access-authorization-details-type)
  - [3.1. Structure of the Authorization Details Object](#31-structure-of-the-authorization-details-object)
  - [3.2. Processing Rules](#32-processing-rules)
- [4. Examples](#4-examples)
- [5. Token Response](#5-token-response)
- [6. Security Considerations](#6-security-considerations)
- [7. Privacy Considerations](#7-privacy-considerations)
- [8. IANA Considerations](#8-iana-considerations)
- [9. Normative References](#9-normative-references)
- [Acknowledgments](#acknowledgments)
- [Authors' Addresses](#authors-addresses)

## 1. Introduction

OAuth 2.0 [RFC6749] and its extension, Rich Authorization Requests (RAR) [RFC9396], provide mechanisms for clients to request fine-grained authorizations from an Authorization Server (AS). Traditionally, these authorizations are tied to an individual user's permissions. However, in collaborative or team-based scenarios, a workload (e.g. an automated service or AI agent) may need to request access on behalf of a group, such as a project team accessing shared resources.

This profile defines a new authorization details type for RAR, enabling clients to request "team access tokens" for workloads. These tokens grant access based on the aggregated permissions of team members, restricted by the workload’s inherent permissions. The aggregation is controlled by an operand: "OR" for the union of all members' permissions (access to any resource any member can access) or "AND" for the intersection (access only to resources all members can access). The access token’s permissions are further limited to those allowed for the workload, ensuring that team permissions do not grant the workload additional privileges (e.g., if the workload cannot delete repositories, it will not gain that ability even if a team member can).

The team identifier and member identifiers are strings, with no specific format required. For clarity, examples in this document use URIs for team identifiers and email addresses for member identifiers, but these are illustrative and not normative. This profile assumes the AS has knowledge of individual user permissions, team memberships, and workload permissions. It is intended for deployments requiring team-based delegation, such as enterprise collaboration tools.

The authorization details type defined here is "urn:ietf:params:oauth:rar:type:team_access" to ensure uniqueness across deployments.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

This specification uses the terms "authorization_details", "Authorization Server" (AS), "Client", "Resource Server" (RS), and other terms defined in [RFC9396] and [RFC6749].

Additional terms:

- **Team**: A group of users identified by a string identifier, with associated member subject identifiers.
- **Operand**: A string value ("AND" or "OR") specifying how to combine permissions across team members.
- **Workload**: An automated service or application identified by a unique identifier, acting on behalf of a team.

## 3. Team Access Authorization Details Type

Clients requesting a team access token for a workload MUST include an authorization details object of type "urn:ietf:params:oauth:rar:type:team_access" in the "authorization_details" parameter of the authorization request or token request, as defined in [RFC9396].

The AS MUST process this type by resolving the permissions of each team member, aggregating them according to the specified operand, and intersecting the result with the workload’s permissions. The resulting access token MUST reflect these constrained permissions, with the `sub` claim set to the workload’s identifier.

If the AS does not have valid consent from each team member, the AS MUST reject the request with an error as per [RFC9396] Section 5. The mechanism by which the AS validates consent is out of scope for this specification.

### 3.1. Structure of the Authorization Details Object

The authorization details object for this type MUST contain the following REQUIRED fields:

- **team**: An object containing team identification and member information:
  - **team_id**: A string identifying the team. This MUST be a unique identifier known to the AS. Examples in this document use URIs (e.g., "https://example.com/teams/avengers") for clarity, but any string format is allowed.
  - **sub_ids**: A JSON array of strings, each representing a subject identifier for a team member. This follows the usage of "sub_ids" in Security Event Token (SET) profiles, such as in RISC events for multiple subjects. Examples in this document use email addresses (e.g., "tony.stark@example.com") for clarity, but any string format is allowed.
- **operand**: A string with value "AND" or "OR":
  - "OR": Aggregate the union of resources accessible by any team member.
  - "AND": Aggregate the intersection of resources accessible by all team members.

The object MAY include additional fields as defined by the AS policy or extensions.

The "sub_ids" field is nested within the "team" object, associating the team members with the team context.

### 3.2. Processing Rules

1. The AS MUST validate that it has consent from each team member identified in the "sub_ids" array within the "team" object.
2. The AS MUST retrieve the individual permissions for each identifier in "sub_ids".
3. The AS MUST aggregate permissions:
   - For "OR": Union of all team members' permissions.
   - For "AND": Intersection of all team members' permissions.
4. The AS MUST intersect the aggregated team permissions with the permissions of the workload (identified in the `sub` claim of the token).
5. If the final permissions are empty (e.g. no overlap between team and workload permissions), the AS MUST reject the request with an `invalid_authorization_details` error.
6. The AS MUST include the processed "authorization_details" in the access token (e.g., as a JWT claim) or introspection response, as per [RFC9396] Section 9.
7. The resulting token's `sub` claim MUST represent the workload.

## 4. Examples

The following example requests a team access token for a workload acting on behalf of the "avengers" team using the "OR" operand:

```json
[
   {
      "type": "urn:ietf:params:oauth:rar:type:team_access",
      "team": {
         "team_id": "https://example.com/teams/avengers",
         "sub_ids": [
            "tony.stark@example.com",
            "steve.rogers@example.com",
            "thor.odinson@example.com",
            "bruce.banner@example.com",
            "natasha.romanoff@example.com"
         ]
      },
      "operand": "OR"
   }
]
```

This requests access to any resource that any Avenger has permission to access, limited to the permissions of the workload.

For "AND":

```json
[
   {
      "type": "urn:ietf:params:oauth:rar:type:team_access",
      "team": {
         "team_id": "https://example.com/teams/avengers",
         "sub_ids": [
            "tony.stark@example.com",
            "steve.rogers@example.com",
            "thor.odinson@example.com",
            "bruce.banner@example.com",
            "natasha.romanoff@example.com"
         ]
      },
      "operand": "AND"
   }
]
```

This requests access only to resources that all Avengers have in common, limited to the permissions of the workload.

## 5. Token Response

The token response follows [RFC9396] Section 7. The "authorization_details" in the response MUST reflect the aggregated permissions, intersected with the workload’s permissions, and potentially enriched or filtered by the AS. The `sub` claim MUST identify the workload.

Example response (abridged):

```json
{
   "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
   "token_type": "Bearer",
   "expires_in": 3600,
   "sub": "spiffe://example.com/workload/jarvis",
   "authorization_details": [
      {
         "type": "urn:ietf:params:oauth:rar:type:team_access",
         "team": {
            "team_id": "https://example.com/teams/avengers",
            "sub_ids": [
               "tony.stark@example.com",
               "steve.rogers@example.com",
               "thor.odinson@example.com",
               "bruce.banner@example.com",
               "natasha.romanoff@example.com"
            ]
         },
         "operand": "OR"
      }
   ]
}
```

## 6. Security Considerations

- **Workload Permission Restriction**: It is the duty of the protected resource to ensure that the workload’s permissions do not exceed those the workload would have when acting on behalf of itself. For example, if the workload is not permitted to delete repositories, it MUST NOT gain this ability even if a team member has such permissions.
- **Permission Aggregation Risks**: The "OR" operand may grant broader access than any one individual has. If the protected resource is shared with the team, team members may gain access that they previously did not have.

## 7. Privacy Considerations

- **Data Aggregation**: Combining permissions may reveal sensitive information about individual members’ accesses.
- **Consent**: If user consent is required, clearly explain the aggregation and workload restriction implications.
- All considerations from [RFC9396] Section 13 apply.

## 8. IANA Considerations

This document has no IANA actions, as the authorization details type is namespaced with a URI under IETF control.

## 9. Normative References

[RFC2119]  
Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/RFC2119, March 1997, <https://www.rfc-editor.org/info/rfc2119>.

[RFC6749]  
Hardt, D., Ed., "The OAuth 2.0 Authorization Framework", RFC 6749, DOI 10.17487/RFC6749, October 2012, <https://www.rfc-editor.org/info/rfc6749>.

[RFC8174]  
Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174, May 2017, <https://www.rfc-editor.org/info/rfc8174>.

[RFC9396]  
Lodderstedt, T., Richer, J., and B. Campbell, "OAuth 2.0 Rich Authorization Requests", RFC 9396, DOI 10.17487/RFC9396, May 2023, <https://www.rfc-editor.org/info/rfc9396>.

## Authors' Addresses

Sarah Cecchetti  
Beyond Identity  

Justin Richer  
MongoDB  

George Fletcher  
