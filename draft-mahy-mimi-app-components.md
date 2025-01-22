---
title: "Application State Components for More Instant Messaging Interoperability (MIMI)"
abbrev: "MIMI Application State"
category: info

docname: draft-mahy-mimi-app-components-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "More Instant Messaging Interoperability"
keyword:
 - participant list
 - roles
 - capabilities
 - room metadata
 - components registry
venue:
  group: "More Instant Messaging Interoperability"
  type: "Working Group"
  mail: "mimi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mimi/"
  github: "rohanmahy/mimi-app-components"
  latest: "https://rohanmahy.github.io/mimi-app-components/draft-mahy-mimi-app-components.html"

author:
 -
    fullname: Rohan Mahy
    organization: Rohan Mahy Consulting Service
    email: rohan.ietf@gmail.com

normative:

informative:


--- abstract

This document presents structures for room metadata, participant lists, pre-authorized roles for future participants, and role-based access control, all of which are intended for use in MIMI (More Instant Messaging Interoperability).

--- middle

# Introduction

This document introduces specific structures to carry room metadata, the room participant list (introduced conceptually in {{!I-D.ietf-mimi-arch}}), room policy for preauthorizing users into the room based on identity-based attributes, and per-room role definitions.
Each of these structures is represented as an MLS application component as defined in {{!I-D.barnes-mls-appsync}} (soon to be merged into {{!I-D.ietf-mls-extensions}}).
Each component is represented in the MLS GroupContext for the room.

This document is provided as a standalone document as it is short and easy to receive early review in its short format. The goal is to incorporate the contents of this draft into {{!I-D.ietf-mimi-room-policy}}.

While this work is intended for use with MIMI, it is suitable for use with
other systems using MLS (for example, non-authority-based messaging systems) that require similar functionality.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Room Metadata

The Room Metadata component contains data about a room which might be displayed as human-readable information for the room, such as the name of the room and a URL pointing to its room image/avatar.

It can contain a list of `room_descriptions`, each of which can have a specific `language_tag` and `media_type` along with the `description_content`. An empty `media_type` implies `text/plain;charset=utf-8`.

RoomMetaData is the format of the `data` field inside the ComponentData struct for the Room Metadata component in the `application_data` GroupContext extension.

~~~ tls-presentation
/* a valid URI (ex: MIMI URI) */
struct {
  opaque uri<V>;
} Uri;

/* a sequence of valid UTF8 without nulls */
struct {
  opaque string<V>;
} UTF8String;

struct {
  /* an empty media_type is equivalent to text/plain;charset=utf-8 */
  opaque media_type<V>;
  opaque language_tag<V>;
  opaque description_content<V>;
} RichDescription;

struct {
  Uri room_uri;
  UTF8String room_name;
  RichDescription room_descriptions<V>;
  /* an https URI resolving to an avatar image */
  Uri room_avatar;
  UTF8String room_subject;
  UTF8String room_mood;
} RoomMetaData;

RoomMetaData RoomMetaUpdate;
~~~

RoomMetaUpdate (which has the same format as RoomMetaData) is the format of the `update` field inside the AppDataUpdate struct in an AppDataUpdate Proposal for the Room Metadata component.
If the contents of the `update` field are valid and if the proposer is authorized to generate such an update, the value of the `update` field completely replaces the value of the `data` field.


# Participant List

The participant list is a list of "users" in a room.
Within a room, each user is assigned exactly one *role* (expressed with a `role_index` and described in {{roles}}) at any given time (specifically within any MLS epoch).
In a room that has multiple MLS clients per "user", the identifier for each user in `participants.user` is the same across all that user's clients in the room.
Note that each user has a single role at any point in time, and therefore all clients of the same user also have the same role.

The participant list may include inactive participants, which currently do not have any clients in the corresponding MLS group, for example if their clients do not have available KeyPackages or if all of their clients are temporarily "kicked" out of the group.
The participant list can also contain participants that are explicitly banned, by assigning them a suitable role which does not have any capabilities.

~~~ tls-presentation
struct {
  opaque user<V>;
  uint32 role_index;
} UserRolePair;

struct {
  UserRolePair participants<V>;
} ParticipantListData;
~~~

ParticipantListData is the format of the `data` field inside the ComponentData struct for the Participant list Metadata component in the `application_data` GroupContext extension.

~~~ tls-presentation
struct {
  uint32 user_index;
  uint32 role_index;
} UserindexRolePair;

struct {
  UserindexRolePair changedRoleParticipants<V>
  uint32 removedIndices<V>;
  UserRolePair addedParticipants<V>;
} ParticipantListUpdate;
~~~

ParticipantListUpdate is the contents of an AppDataUpdate Proposal with the component ID for the participant list.
The index of the `participants` vector in the current `ParticipantListData` struct is referenced as the `user_index ` when making changes.
First the `changedRoleParticipants` list contains `UserindexRolePair`s with the index of a user who changed roles and their new role.
Next is the `removedIndices` list which has a list of users to remove completely from the participant list.
Finally there is a list of `addedParticipants` (which contains a user and role) that is appended to the end of the `ParticipantListData`.

Each of these actions (modifying a user's role, removing a user, and adding a user) is authorized separately according to the rules specified in {{membership-capabilities}}. If all the changes are authorized, the `ParticipantListData` is modified accordingly.

A single commit is not valid if it contain any combination of Participant list updates that operate on (add, remove, or change the role of) the same user in the participant list more than once.


# Preauthorized Users

Preauthorized users are MIMI users and external senders that have authorization to adopt a role in a room by virtue of certain credential claims or properties, as opposed to being individually enumerated in the participant list.
For example, a room for employee benefits might be available to join with the regular participant role to all full-time employees with a residence in a specific country; while anyone working in the human resources department might be able to join the same room as a moderator.
This data structure is consulted in two situations: for external joins (external commits) and external proposals when the requester does not already appear in the participant list; and separately when an existing participant explicitly tries to change its *own* role.

>Only consulting Preauthorized users in these cases prevents several attacks. For example, it prevents an explicitly banned user from rejoining a group based on a preauthorization.

PreAuthData is the format of the `data` field inside the ComponentData struct for the Preauthorized Participants component in the `application_data` GroupContext extension.

The individual `PreAuthRoleEntry` rules in `PreAuthData` are consulted one at a time.
A `PreAuthRoleEntry` matches for a requester when every `Claim.claim_id` has a corresponding claim in the requester's MLS Credential which exactly matches the corresponding `claim_value`.
When the rules in a Preauthorized users struct match multiple roles, the requesting client receives the first role which matches its claims.


~~~ tls-presentation
struct {
  /* MLS Credential Type of the "claim"  */
  CredentialType credential_type;
  /* the binary representation of an X.509 OID, a JWT claim name  */
  /* string, or the CBOR map claim key in a CWT (an int or tstr)  */
  opaque id<V>;
} ClaimId;

struct {
  ClaimId claim_id;
  opaque claim_value<V>;
} Claim;

struct {
  /* when all claims in the claimset are satisfied, the claimset */
  */ is satisfied */
  Claim claimset<V>;
  Role target_role;
} PreAuthRoleEntry;

struct {
  PreAuthRoleEntry preauthorized_entries<V>;
} PreAuthData;

PreAuthData PreAuthUpdate;
~~~

<!--
struct {
  select (Credential.credential_type) {
    case basic:
        struct {}; /* only identity */
    case x509:
        /* ex: subjectAltName (2.5.29.17) = hex 06 03 55 1d 1e */
        opaque oid<V>;
        /* for sequence or set types, the specific item (1-based) */
        /* in the collection. zero means any item in a collection */
        uint8 ordinal;
    case jwt:
        opaque json_path<V>;
    case cwt:
        CborKeyNameOrArrayIndex cbor_path<V>;
  };
} Claim;

struct {
    /* a CBOR CDE encoded integer, tstr, bstr, or tagged version of */
    /* any of those map key types. Ex: -1 = 0x20, "hi" = 0x626869,  */
    /* 1(3600) = 0xC1190E10 */
    opaque cbor_encoded_claim<V>;
    optional uint array_index;
} CborKeyNameOrArrayIndex;
-->

PreAuthUpdate (which has the same format as PreAuthData) is the format of the `update` field inside the AppDataUpdate struct in an AppDataUpdate Proposal for the Preauthorized Participants component.
If the contents of the `update` field are valid and if the proposer is authorized to generate such an update, the value of the `update` field completely replaces the value of the `data` field.

>As with the definition of roles, in MIMI it is not expected that the definition of Preauthorized users would change frequently. Instead the claims in the underlying credentials would be modified without modifying the preauthorization policy.

Changing Preauthorized user definitions is sufficiently disruptive, that an update to this component is not valid if it appears in the same commit as any Participant List change, except for user removals.

Because the Preauthorized users component usually authorizes non-members, it is also a natural choice for providing concrete authorization for policy enforcing systems incorporated into or which run in coordination with the MIMI Hub provider or specific MLS Distribution Services. For example, a preauthorized role could allow the Hub to remove participants and to ban them, but not to add any users or devices. This unifies the authorization model for members and non-members.


# Role-Based Access Control {#roles}

The Role-Based Access Control component contains a list of all the roles in the room, and the capabilities associated with them.
It contains a `role_index`, which is used to refer to the role elsewhere. (Note that role indexes might not be contiguous.)
The `role_index` zero is reserved to refer to a participant that does not (yet) or no longer appears (or will no longer appear) in the participant list.

The component also contains a `role_name` (a human-readable text string name for the
role), and a `role_description` (another string, which can have zero length).

Each Role also can contain constraints on the minimum and maximum number of participants, and the minimum and maximum number of active participants.
If the minimum number is zero, there is no minimum number of participants for that particular role.
If there is no maximum number of participants for a particular role, that parameter is absent.

>If the maximum number of active participants is zero, then no participants are allowed to have clients in the room's MLS group.

The `authorized_role_changes` field is used to provide fine-grained control about which transitions are allowed when adding and removing participants and when moving participants to new roles, including banning/unbanning, and promoting/demoting to or from roles with moderator or administrator privileges.
A more detailed discussion is in the description of the specific capabilities in the next section.

>This design results in each participant only having a single role at a time, with a single list of capabilities and an explicit list of allowed role transitions. It makes the authorization process for a verifier consistent regardless of the complexity of the set of authorization rules.

Some examples are provided in {{role-examples}}.

RoleData is the format of the `data` field inside the ComponentData struct for the Role-Based Access Control component in the `application_data` GroupContext extension.

~~~ tls-presentation
/* See MIMI Capability Types IANA registry */
uint16 CapablityType;

struct {
   uint32 from_role_index;
   uint32 target_role_indexes<V>;
} SingleSourceRoleChangeTargets;

struct {
  uint32 role_index;
  opaque role_name<V>;
  opaque role_description<V>;
  CapabilityType role_capabilities<V>;
  uint32 minimum_participants_constraint;
  optional uint32 maximum_participants_constraint;
  uint32 minimum_active_participants_constraint;
  optional uint32 maximum_active_participants_constraint;
  SingleSourceRoleChangeTargets authorized_role_changes<V>;
} Role;

struct {
  Role roles<V>;
} RoleData;

RoleData RoleUpdate;
~~~

RoleUpdate (which has the same format as RoleData) is the format of the `update` field inside the AppDataUpdate struct in an AppDataUpdate Proposal for the Role-Based Access Control component.
If the contents of the `update` field are valid and if the proposer is authorized to generate such an update, the value of the `update` field completely replaces the value of the `data` field.

>Note that in the MIMI environment, changing the definitions of roles is anticipated to be very rare over the lifetime of a room (for example changing a room which has grown dramatically from cooperatively managed by all participants to explicitly moderated or administered).

Changing Role definitions is sufficiently disruptive, that an update to this component is not valid if it appear in the same commit as any Participant List change.

# Role Capabilities

As described in the previous section, each role has a list of capabilities, which in rare cases could be empty.
When we say that the holder of a capability can take some action, we mean that whatever entity is taking the action (a participant, a potential future participant, or an external party) has a specific entry in the Participant List struct and a corresponding role--or is preauthorized to take action with a specific role via the Preauthorized Users struct--and that the `role_capabilities` list contains the relevant capability.

Unless otherwise specified, capabilities apply both to sending a set of consistent MLS proposals that could be committed by any member of the corresponding MLS group, and to sending an MLS commit containing a set of consistent MLS proposals.

## Membership Capabilities

The membership capabilities below allow authorized holders to update the Participant list, or change the active participants (by removing and adding MLS clients corresponding to those participants), or both.

- `canAddParticipant` - the holder of this capability can add another user, that is not already in the participant list, to the participant list.
  (This capability does not apply to the holder adding itself.)
  The `authorized_role_changes` list in the holder's role is consulted to authorize the added user's target role.
  The `authorized_role_changes` list MUST have an entry where the `authorized_role_changes.from_role_index` equals zero, and that entry's `target_role_indexes` list includes the target role.
  The proposed action is only authorized if the action respects both the `maximum_participants_constraint` (if present) and `maximum_active_participants_constraint` (if present) for the added user's target role.
  When the participant list addition for the target role is authorized, the holder is also authorized to add any MLS clients matching the added user to the room's MLS group .

- `canAddOwnClient` - a holder of this capability that is in the participant list, can add its own client (via an external commit or external proposal); and can add other clients that share the same user identity (via Add proposals) if the holder's client is already a member of the corresponding MLS group.

  >**TODO**: consider if multiple clients per user are allowed; client replacements.

- `canAddSelf` - the holder of this capability can use an external commit or external proposal to add itself to the participant list.
  (The holder MUST NOT already appear in the participant list).
  Its usage differs slightly based on in which role it appears.
  - When `canAddSelf` appears on role zero, any user who is not already in the participant list can add itself, with certain provisions. The holder consults the `authorized_role_changes` list for an entry with `from_role_index` equal to zero. The holder can add itself with any non-zero `target_role_indexes` from that entry, if the action respects both the `maximum_participants_constraint` (if present) and `maximum_active_participants_constraint` (if present) for the added user's target role.
  - When `canAddSelf` appears on a non-zero role, a client can only become the holder of this capability via the Preauthorized users mechanism.
    The `authorized_role_changes` list in the target role MUST have an entry where the `from_role_index` is zero and the `target_role_indexes` contains the target role.
    In addition, the action MUST respect both the `maximum_participants_constraint` (if present) and `maximum_active_participants_constraint` (if present) for the added user's target role.

- `canUseJoinCode` - the holder of this capability can externally join a room using a join code for that room, provided the join code is valid, the join code refers to a valid target role, and both the `maximum_participants_constraint` (if present) and `maximum_active_participants_constraint` (if present) constraints are respected.


- `canRemoveParticipant` - the holder of this capability can propose a) the removal of another user (excluding itself) from the participant list, and b) removal of all of that user's clients, as a single action.
  There MUST NOT be any clients of the removed user in the MLS group after the corresponding commit.
  A proposer holding this capability consults its role's `authorized_role_changes` entries for an entry where `from_role_index` matches the target user's current role; if the `target_role_indexes` for that entry contains zero, and the `minimum_participants_constraint` and `minimum_active_participants_constraint` are satisfied, the proposal is authorized.

- `canRemoveOwnClient` - the holder of this capability can propose to remove its own client using an MLS Remove or SelfRemove proposal without changing the Participant list.
  Due to restrictions in MLS which insure the consistency of the group, this action cannot be committed by the leaving user.
  If the `minimum_active_participants_constraint` is satisfied, the proposal is authorized.

- `canRemoveSelf` - the holder of this capability can propose to remove itself from (i.e. leave) the participant list; it MUST simultaneously propose to remove all of its remaining clients from the corresponding MLS group.
  Due to restrictions in MLS which insure the consistency of the group, this action cannot be committed by the leaving user.
  A proposer holding this capability consults its role's `authorized_role_changes` entries for an entry where `from_role_index` matches its current role; if the `target_role_indexes` for that entry contains zero, and the `minimum_participants_constraint` and `minimum_active_participants_constraint` are satisfied, the proposal is authorized.

- `canKick` - the holder of this capability can propose removal of another participant's clients, without changing the Participant List.
  If the `minimum_active_participants_constraint` is satisfied, the proposal is authorized.


- `canChangeUserRole` - the holder of this capability is authorized to change the role of another participant (but not itself), according to the holder's `authorized_role_changes` list, from a role represented by an entry where the target's current role matches `from_role_index` to any of the non-zero `target_role_indexes` in the same element of `authorized_role_changes`.
  The `minimum_participants_constraint` and `minimum_active_participants_constraint` for the target user's current role, and the `maximum_participants_constraint` (if present) and `maximum_active_participants_constraint` (if present) for the target user's target role must also be satisfied.

- `canChangeOwnRole` - the holder of this capability is authorized to change its own role to the first non-zero role it matches in the Preauthorized users component (see {{preauthorized-users}}).
  The `authorized_role_changes` list is *not* consulted.
  The `minimum_participants_constraint` and `minimum_active_participants_constraint` for the holder's original role, and the
`maximum_participants_constraint` (if present) and `maximum_active_participants_constraint` (if present) for the holder's target role must also be satisfied.

- `canBan` - the holder of this capability can propose to "ban" another user.
  Specifically, a successful ban changes the target user's role to a special "banned" role (if it exists), and removes all the banned user's clients.
  The "banned" role always has `role_index` = 1 and `role_name` = "banned" (without quotes).

  > A "banned" role does not have to exist in a room, but to use the `canBan` and `canUnban` capabilities, the role needs to exist exactly as described above.
  > While holding `canChangeUserRole` and `canKick` capabilities would allow the same action, it could potentially allow the holder other actions which might be undesirable in some contexts, such as kicking clients without banning.

  A proposer holding this capability consults its role's `authorized_role_changes` entries for an entry where `from_role_index` matches the target user's current role; if the `target_role_indexes` for that entry contains the `role_index` 1; that `role_name` = "banned" for the role with role_index = 1, and the `minimum_participants_constraint` and `minimum_active_participants_constraint` are satisfied, the proposal is authorized.

- `canUnban` - the holder of this capability can propose to "unban" another user.
  Specifically, a successful unban changes the target user's role from `role_index` = 1 to another non-zero `role_index` allowed by the holder's `authorized_role_changes` list.
  Adding clients for that unbanned user is *not* authorized by this capability.
  The authorization of this capability is identical to the `canChangeUserRole` capability, except that the `from_role_index` for the unbanned user MUST be 1, and the `role_name` of role 1 MUST be "banned".


## Adjust metadata

- canChangeRoomName
- canChangeRoomDescription
- canChangeRoomAvatar
- canChangeRoomSubject
- canChangeRoomMood
- canChangeOwnName
- canChangeOwnPresence
- canChangeOwnMood
- canChangeOwnAvatar


## Message Capabilities

The capabilities below refer to functionality related to the instant messages, for example sent using the MIMI content format {{!I-D.ietf-mimi-content}}.

- canSendMessage
- canReceiveMessage
- canReportAbuse
- canReactToMessage
- canEditReaction
- canDeleteReaction
- canEditOwnMessage
- canDeleteOwnMessage
- canDeleteAnyMessage
- canStartTopic
- canReplyInTopic
- canEditTopic
- canSendLink
- canSendLinkPreview
- canFollowLink

The Hub can enforce whether a member can send a message. It can also withhold fanout of application messages to clients of a user. The other capabilities in this section can only be enforced by other clients.


## Asset Capabilities

- canUploadImage
- canDownloadImage
- canUploadVideo
- canDownloadVideo
- canUploadSound
- canDownloadSound
- canUploadAttachment
- canDownloadAttachment


## Real-time media

- canStartCall
- canJoinCall
- canSendAudio
- canReceiveAudio
- canSendVideo
- canReceiveVideo
- canShareScreen
- canViewSharedScreen

## Disruptive Policy Changes

- canChangeRoomMembershipStyle
- canChangeRoleDefinitions
- canChangePreauthorizedUserList
- canChangeOtherPolicyAttribute
- canCreateRoom - reserved
- canDestroyRoom
- canReinitGroup


## Reserved Capabilities

The following capability names are reserved for possible future use

- `canCreateJoinCode`
- `canKnock`
- `canAcceptKnock`
- `canCreateSubgroup`
- `canSendDirectMessage`
- `canTargetMessage`
- MLS specific
  - update - update policy
  - PSK - psk policy
  - external proposal - general operational policy rules
  - external commit - general operational policy rules


# Security Considerations

TODO Security


# IANA Considerations

## New MIMI Role Capabilities registry

Create a new registry with the following values assigned sequentially using the reference RFCXXXX.

~~~
canAddParticipant
canRemoveParticipant
canAddOwnClient
canRemoveSelf
canAddSelf
canCreateJoinCode - reserved for future use
canUseJoinCode
canBan
canUnBan
canKick
canKnock
canAcceptKnock
canChangeUserRole
canChangeOwnRole
canCreateSubgroup
canSendMessage
canReceiveMessage
canReportAbuse
canReactToMessage
canEditReaction
canDeleteReaction
canEditOwnMessage
canDeleteOwnMessage
canDeleteAnyMessage
canStartTopic
canReplyInTopic
canEditTopic
canSendDirectMessage
canTargetMessage
canUploadImage
canUploadVideo
canUploadAttachment
canDownloadImage
canDownloadVideo
canDownloadAttachment
canSendLink
canSendLinkPreview
canFollowLink
canChangeRoomName
canChangeRoomDescription
canChangeRoomAvatar
canChangeRoomSubject
canChangeRoomMood
canChangeOwnName
canChangeOwnPresence
canChangeOwnMood
canChangeOwnAvatar
canStartCall
canJoinCall
canSendAudio
canReceiveAudio
canSendVideo
canReceiveVideo
canShareScreen
canViewSharedScreen
canChangeRoomMembershipStyle
canChangeRoleDefinitions
canChangePreauthorizedUserList
canChangeOtherPolicyAttribute
canDestroyRoom
canSendMLSUpdateProposal
canSendMLSReinitProposal
canSendMLSPSKProposal
canSendMLSExternalProposal
canSendMLSExternalCommit
~~~

--- back

# Role examples

## Cooperatively administered room

This is an example set of role policies, which is suitable for friends and family rooms and small groups of peers in a workgroup or club.

- no_role
   - role_index = 0
   - no capabilities
   - constraints
      - minimum_participants_constraint = 0
      - maximum_participants_constraint = null
      - minimum_active_participants_constraint = 0
      - maximum_active_participants_constraint = null
      - authorized_role_changes = `[]`

- banned
   - role_index = 1
   - no capabilities
   - constraints
      - minimum_participants_constraint = 0
      - maximum_participants_constraint = null
      - minimum_active_participants_constraint = 0
      - maximum_active_participants_constraint = null
      - authorized_role_changes = `[]`

- ordinary_user
   - role_index = 2
   - authorized capabilities
      - canAddParticipant
      - canRemoveParticipant
      - canAddOwnClient
      - canRemoveSelf
      - canAddSelf
      - canCreateJoinCode - reserved for future use
      - canUseJoinCode
      - canKnock
      - canSendMessage
      - canReceiveMessage
      - canReportAbuse
      - canReactToMessage
      - canEditReaction
      - canDeleteReaction
      - canEditOwnMessage
      - canDeleteOwnMessage
      - canStartTopic
      - canReplyInTopic
      - canUploadImage
      - canUploadVideo
      - canUploadAttachment
      - canDownloadImage
      - canDownloadVideo
      - canDownloadAttachment
      - canSendLink
      - canSendLinkPreview
      - canFollowLink
      - canChangeRoomName
      - canChangeRoomAvatar
      - canChangeRoomSubject
      - canChangeRoomMood
      - canChangeOwnName
      - canChangeOwnPresence
      - canChangeOwnMood
      - canChangeOwnAvatar
   - constraints
      - minimum_participants_constraint = 0
      - maximum_participants_constraint = null
      - minimum_active_participants_constraint = 0
      - maximum_active_participants_constraint = null
      - authorized_role_changes = `[(0,[2]), (2,[0])]`

- group_admin
   - role_index = 3
   - authorized capabilities
      - (include all the capabilities authorized for an ordinary_user)
      - canBan
      - canUnBan
      - canKick
      - canRevokeVoice
      - canGrantVoice
      - canAcceptKnock
      - canChangeUserRole
      - canDeleteAnyMessage
      - canEditTopic
      - (canDeleteUpload)
      - canChangeRoomDescription
   - constraints
      - minimum_participants_constraint = 1
      - maximum_participants_constraint = null
      - minimum_active_participants_constraint = 0
      - maximum_active_participants_constraint = null
      - authorized_role_changes = `[(0,[1,2,3]), (1,[0,2,3]), (2,[0,1,3]), (3,[0,1,2])]`

- super_admin
   - role_index = 4
   - authorized capabilities
      - (include all the capabilities authorized for a group_admin)
      - canChangeRoomMembershipStyle
      - canChangePreauthorizedUserList
      - canChangeOtherPolicyAttribute
      - canDestroyRoom
   - constraints
      - minimum_participants_constraint = 0
      - maximum_participants_constraint = null
      - minimum_active_participants_constraint = 0
      - maximum_active_participants_constraint = null
      - authorized_role_changes = `[(0,[1,2,3,4]), (1,[0,2,3,4]), (2,[0,1,3,4]), (3,[0,1,2,4]), (4,[0,1,2,3])]`


- policy_enforcer
   - role_index = 5
   - capabilities
      - (does not include any other capabilities)
      - canRemoveParticipant
      - canChangeUserRole
      - canBan
      - canUnban ??
      - canChangeRoomMembershipStyle
      - canChangeRoleDefinitions
      - canChangePreauthorizedUserList
      - canChangeOtherPolicyAttribute
      - canDestroyRoom
      - canSendMLSReinitProposal
      - canSendMLSExternalProposal
   - constraints
      - minimum_participants_constraint = 1
      - maximum_participants_constraint = 2
      - minimum_active_participants_constraint = 0
      - maximum_active_participants_constraint = 0
      - authorized_role_changes = `[(0,[1]), (1,[0]), (2,[0,1]), (3,[0,1]), (4,[0,1])]`
   - Notes: can remove a banned user from the list (cleanup) but not restore them



## Strictly administered room


## Moderated room


## Multi-organization administered room




# Acknowledgments
{:numbered="false"}

TODO acknowledge.
