```markdown
# OAuth 2.0 Rich Authorization Requests Profile: Team Access Tokens

## Abstract

This document defines a profile of OAuth 2.0 Rich Authorization Requests (RAR) [RFC9396] for requesting access tokens that grant permissions based on the collective access rights of a team, rather than an individual user. It introduces a new authorization details type that allows clients to specify a team identifier, its members (using "sub_ids" as an array of subject identifiers, consistent with usage in Security Event Token profiles such as RISC events), and an operand ("AND" or "OR") to determine how individual team member permissions are combined (intersection or union). This enables scenarios where applications need to act on behalf of a group, such as shared resource access in collaborative environments.

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

OAuth 2.0 [RFC6749] and its extension, Rich Authorization Requests (RAR) [RFC9396], provide mechanisms for clients to request fine-grained authorizations from an Authorization Server (AS). Traditionally, these authorizations are tied to an individual user's permissions. However, in collaborative or team-based scenarios, applications may need to request access on behalf of a group (e.g., a project team accessing shared resources).

This profile defines a new authorization details type for RAR, enabling clients to request "team access tokens." These tokens grant access based on the aggregated permissions of team members. The aggregation is controlled by an operand: "OR" for the union of all members' permissions (access to any resource any member can access) or "AND" for the intersection (access only to resources all members can access).

This profile assumes the AS has knowledge of individual user permissions and can resolve team memberships. It is intended for deployments where team-based delegation is required, such as enterprise collaboration tools.

The authorization details type defined here is "urn:ietf:params:oauth:rar:type:team_access" to ensure uniqueness across deployments.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

This specification uses the terms "authorization_details", "Authorization Server" (AS), "Client", "Resource Server" (RS), and other terms defined in [RFC9396] and [RFC6749].

Additional terms:

- **Team**: A group of users identified by a team identifier, with associated member subject identifiers.
- **Operand**: A string value ("AND" or "OR") specifying how to combine permissions across team members.

## 3. Team Access Authorization Details Type

Clients requesting a team access token MUST include an authorization details object of type "urn:ietf:params:oauth:rar:type:team_access" in the "authorization_details" parameter of the authorization request or token request, as defined in [RFC9396].

The AS MUST process this type by resolving the permissions of each team member and aggregating them according to the specified operand. The resulting access token MUST reflect the combined permissions.

If the requester (e.g., the authenticated user) is not authorized to request on behalf of the team (e.g., not a team member or administrator), the AS MUST reject the request with an error as per [RFC9396] Section 5.

### 3.1. Structure of the Authorization Details Object

The authorization details object for this type MUST contain the following REQUIRED fields:

- **team_id**: A string identifying the team (e.g., "avengers"). This MUST be a unique identifier known to the AS.
- **sub_ids**: A JSON array of strings, each representing a subject identifier (sub) for a team member. This follows the usage of "sub_ids" in Security Event Token (SET) profiles, such as in RISC events for multiple subjects.
- **operand**: A string with value "AND" or "OR":
  - "OR": Grant access to the union of resources accessible by any team member.
  - "AND": Grant access only to the intersection of resources accessible by all team members.

The object MAY include additional fields as defined by the AS policy or extensions.

The "sub_ids" field nests the team members under the team context, as they are associated with the "team_id".

### 3.2. Processing Rules

1. The AS MUST validate the "team_id" and confirm that the provided "sub_ids" match the team's membership (as known to the AS).
2. The AS MUST retrieve the individual permissions for each sub in "sub_ids".
3. The AS MUST aggregate permissions:
   - For "OR": Union of all permissions.
   - For "AND": Intersection of all permissions.
4. If aggregation results in no permissions, the AS SHOULD reject the request.
5. The AS MUST include the processed "authorization_details" in the access token (e.g., as a JWT claim) or introspection response, as per [RFC9396] Section 9.
6. The resulting token's "sub" claim MAY represent the team (e.g., "team:avengers") or the requester, with the team details in "authorization_details".

## 4. Examples

The following example requests a team access token for the "avengers" team using the "OR" operand:

```json
[
   {
      "type": "urn:ietf:params:oauth:rar:type:team_access",
      "team_id": "avengers",
      "sub_ids": [
         "tony_stark",
         "steve_rogers",
         "thor_odinson",
         "bruce_banner",
         "natasha_romanoff"
      ],
      "operand": "OR"
   }
]
```

This requests access to any resource that any Avenger has permission to access.

For "AND":

```json
[
   {
      "type": "urn:ietf:params:oauth:rar:type:team_access",
      "team_id": "avengers",
      "sub_ids": [
         "tony_stark",
         "steve_rogers",
         "thor_odinson",
         "bruce_banner",
         "natasha_romanoff"
      ],
      "operand": "AND"
   }
]
```

This requests access only to resources that all Avengers have in common.

## 5. Token Response

The token response follows [RFC9396] Section 7. The "authorization_details" in the response MUST reflect the aggregated permissions, potentially enriched or filtered by the AS.

Example response (abridged):

```json
{
   "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
   "token_type": "Bearer",
   "expires_in": 3600,
   "authorization_details": [
      {
         "type": "urn:ietf:params:oauth:rar:type:team_access",
         "team_id": "avengers",
         "sub_ids": ["tony_stark", "steve_rogers", "thor_odinson", "bruce_banner", "natasha_romanoff"],
         "operand": "OR"
      }
   ]
}
```

## 6. Security Considerations

- **Team Membership Validation**: The AS MUST verify team membership to prevent unauthorized inclusion of subjects.
- **Permission Aggregation Risks**: "OR" may grant broader access than intended; "AND" may be too restrictive. Clients SHOULD use the minimal operand needed.
- **Requester Authorization**: The AS MUST ensure the requester (client or user) has authority to request team access (e.g., team admin).
- **Token Binding**: Use token binding mechanisms to prevent token theft, as team tokens may have elevated privileges.
- All considerations from [RFC9396] and [RFC6749] apply.

## 7. Privacy Considerations

- **Data Aggregation**: Combining permissions may reveal sensitive information about individual members' accesses.
- **Minimal Disclosure**: The AS SHOULD minimize exposed details in tokens and introspection responses.
- **Consent**: If user consent is required, clearly explain the aggregation implications.
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

## Acknowledgments

The authors thank the contributors to RFC 9396, as this profile builds upon it.

## Authors' Addresses

Sarah Cecchetti  
Beyond Identity  
Email: sarah@beyondidentity.com  

Justin Richer  
MongoDB  
Email: justin.richer@mongodb.com  

George Fletcher  
Email: george.fletcher@example.com
```