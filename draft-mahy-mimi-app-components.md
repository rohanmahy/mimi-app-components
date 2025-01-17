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

This document presents structures for room metadata, role-based access control, participant lists, and pre-authorized roles for future participants intended for use in MIMI (More Instant Messaging Interoperability).

--- middle

# Introduction

This document is provided as a standalone document as it is short and easy to receive early review in its short format. The goal is to incorporate the contents of this draft into {{!I-D.ietf-mimi-room-policy}}.

While this work is intended for use with MIMI, it is suitable for use with
other systems using MLS (for example, non-authority-based messaging systems) that require similar functionality.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Room Metadata

The Room Metadata component contains data about a room which might be displayed as human-readable information for the room, such as the name of a room and URL point to its room image/avatar.

It can contain a list of `room_descriptions`, each of which can have a specific `language_tag` and `media_type` along with the `description_content`. An empty `media_type` implies `text/plain;charset=utf-8`.

RoomMetaData is the format of the `data` field inside the ComponentData struct for the Room Metadata component in the `application_data` GroupContext extension.

~~~ tls
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
} RichDescription

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

RoomMetaUpdate (which has the same format as RoomMetaData) is the format of the `update` field inside the ApplicationDataUpdate struct in an ApplicationDataUpdate Proposal for the Room Metadata component.
If the contents of the `update` field are valid and if the proposer is authorized to generate such an update, the value of the `update` field completely replaces the value of the `data` field.

# Role-Based Access Control

The Role-Based Access Control component contains a list of all the roles in the room, and the capabilities associated with them.
It contains a `role_index`, which is used to refer to the role elsewhere. (Note that role indexes might not be contiguous.)
The `role_index` zero is reserved to refer to a participant that does not (yet) or no longer appears (or will no longer appear) in the participant list.

The component also contains a `role_name` (a human-readable text string name for the
role), and a `role_description` (another string, which can have zero length).

Each Role also can contain constraints on the minimum and maximum number of participants, and the minimum and maximum number of active participants.
If the minimum number is zero, there is no minimum number of participants for that particular role.
If there is no maximum number of participants for a particular role, that parameter is absent.

>If the maximum number of active participants is zero, then no participants are allowed to have clients in the room's MLS group.

A party with a particular Role which has the `canAddParticipant` capability is authorized to add another (new) participant with any of the `target_role_indexes` in an `authorized_role_changes` entry where the `authorized_role_changes.from_role_index` equals zero. It is also authorized to add any MLS clients matching an authorized added user to the room's MLS group.
A party with a particular Role which has the `canRemoveParticipant` capability is authorized to remove another participant when the target user's role matching `authorized_role_changes.from_role_index` contains zero in the `target_role_indexes`. It MUST also remove any and all clients belonging to a removed user in the same commit.

A party with a particular Role which has the `canChangeUserRole` capability is authorized to change the role of another participant (but not itself) from a role represented by `authorized_role_changes.from_role_index` to any of the `target_role_indexes` in the same element of `authorized_role_changes`.

A party with a particular Role which has the `canChangeOwnRole` can change its own role to the first role matching in the Preauthorized users component (see {{preauthorized-users}}).


>This design results in each participant only having a single role at a time, with a single list of capabilities and an explicit list of allowed role transitions. It makes the authorization process for a verifier consistent regardless of the complexity of the set of authorization rules.

Some examples are provided in {{role-examples}.

RoleData is the format of the `data` field inside the ComponentData struct for the Role-Based Access Control component in the `application_data` GroupContext extension.

~~~ tls
/* See MIMI Capability Types IANA registry */
uint16 CapablityType;

struct {
   uint32 from_role_index;
   uint32 target_role_indexes<V>;
} SingleSourceRoleChangeTargets;

struct {
  int role_index;
  opaque role_name<V>;
  opaque role_description<V>;
  CapabilityType role_capabilities<V>;
  int minimum_participants_constraint;
  optional int maximum_participants_constraint;
  int minimum_active_participants_constraint;
  optional int maximum_active_participants_constraint;
  SingleSourceRoleChangeTargets authorized_role_changes<V>;
} Role;

struct {
  Role roles<V>;
} RoleData;

RoleData RoleUpdate;
~~~

RoleUpdate (which has the same format as RoleData) is the format of the `update` field inside the ApplicationDataUpdate struct in an ApplicationDataUpdate Proposal for the Role-Based Access Control component.
If the contents of the `update` field are valid and if the proposer is authorized to generate such an update, the value of the `update` field completely replaces the value of the `data` field.

>Note that in the MIMI environment, changing the definitions of roles is anticipated to be very rare over the lifetime of a room (for example changing a room which has grown dramatically from cooperatively managed by all participants to explicitly moderated or administered).

# Participant List

The participant list is a list of "users" in a room.
Each user is assigned one role (expressed as the role_index) in the room.
In a room that has multiple MLS clients per "user", the identifier for each user in `participants.user` is the same across all that user's clients in the room.
Note that each user has a single role at any point in time.

The participant list may include inactive participants, which currently do not have any clients in the corresponding MLS group, for example if their clients do not have available KeyPackages or if all of their clients are temporarily "kicked" out of the group.
The participant list can also contain participants that are explicitly banned, by assigning them a suitable role which does not have any capabilities.

~~~ tls
struct {
  opaque user<V>;
  int role_index;
} UserRolePair;

struct {
  uint32 user_index;
  int role_index;
} UserindexRolePair

struct {
  UserRolePair participants<V>;
} ParticipantListData;

struct {
  uint32 removedIndices<V>;
  UserindexRolePair changedRoleParticipants<V>
  UserRolePair addedParticipants<V>;
} ParticipantListUpdate;
~~~

ParticipantListUpdate is the contents of an ApplicationDataUpdate Proposal with the component ID for the participant list.


# Preauthorized Users

Preauthorized users are MIMI users and external senders that have authorization to adopt a role in a room by virtue of certain credential claims or properties, as opposed to being individually enumerated in the participant list.
For example, a room for employee benefits might be available to join with the regular participant role to all employees with a residence in a specific country; anyone working in the human resources department might be able to join the same room as a moderator.

PreAuthData is the format of the `data` field inside the ComponentData struct for the Preauthorized Participants component in the `application_data` GroupContext extension.

The Preauthorized users data structure is used to authorize external joins (external commits) and external proposals, and only when the requester does not already appear in the participant list. This prevents an explicitly banned user from rejoining a group based on a preauthorization.

When the rules in a Preauthorized users struct match multiple roles, the requesting client receives the first role which matches its claims.

~~~ tls
struct {
  /* MLS Credential Type of the "claim"  */
  CredentialType credential_type;
  /* the binary representation of an X.509 OID, a JWT claim name  */
  /* string, or the value inside the CBOR representation of a CWT */
  /* claim (an int or tstr) */
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

PreAuthUpdate (which has the same format as PreAuthData) is the format of the `update` field inside the ApplicationDataUpdate struct in an ApplicationDataUpdate Proposal for the Preauthorized Participants component.
If the contents of the `update` field are valid and if the proposer is authorized to generate such an update, the value of the `update` field completely replaces the value of the `data` field.

>As with the definition of roles, in MIMI it is not expected that the definition of Preauthorized users would change frequently. Instead the claims in the underlying credentials would be modified without modifying the preauthorization policy.

# Role Capabilities

When we say that the holder of this capability can take some action, we mean that the whatever entity is taking the action (a participant, a potential future participant, or an external party) has a specific role already pre-assigned in the Participant List struct, or is pre-authorized to take action with a specific role in the Preauthorized Users struct.

Unless otherwise specified, capabilities apply both to sending a set of consistent MLS proposals that could be committed by any member of the corresponding MLS group, and to sending an MLS commit containing a set of consistent MLS proposals.

## Membership Capabilities



- `canAddParticipant` - the holder of this capability can add another user to the participant list. (This capability does not apply to the holder adding itself.) The holder may assign the target user any role in the `add_participant_role_indexes` list of the holder's Role. The proposed action is only authorized if the action respects both the `maximum_participants_constraint` (if present) and `maximum_active_participants_constraint` (if present) for the added user's target role.
- `canRemoveParticipant` - the holder of this capability can propose the removal of another user (excluding itself) from the participant list and any of their clients. There MUST NOT be any clients of the removed user in the MLS group after the corresponding commit. The proposed action is only authorized if the action respects both the `minimum_participants_constraint` and `minimum_active_participants_constraint` for the removed user's role. **TODO**: users can be removed with which roles? can you remove a banned user or a superadmin.
- `canAddOwnClient` - the holder of this capability can add its own client (via an external commit or external proposal) and may add other clients that share the same user identity (via Add proposals) if the corresponding user is already in the participant list. **TODO**: consider if multiple clients per user are allowed; client replacements.
- `canRemoveSelf` - the holder of this capability can propose to remove itself from (i.e. leave) the participant list; it MUST simultaneously propose to remove all of its remaining clients from the corresponding MLS group. Due to restrictions in MLS which insure the consistency of the group, this action cannot be committed by the leaving user.
- `canAddSelf` - the holder of this capability can use an external commit or external proposal to add itself to the participant list, and to the room, when none of it's clients are present. The proposed action is only authorized if the action respects both the `maximum_participants_constraint` (if present) and `maximum_active_participants_constraint` (if present) for the added user's target role.
- `canCreateJoinCode` - the holder of this capability can
- `canUseJoinCode` - the holder of this capability can join a room using a valid join code for that room, provided the join code is valid, and the rest of its constraints are satisfied.


## Moderation Capabilities

- `canKick` - the holder of this capability can remove all the clients of a user (excluding itself), without removing the user from the participation list.
- `canChangeUserRole` - the holder of this capability can change the role of another user from
- `canBan` - the holder of this capability can
- `canUnBan` - the holder of this capability can
- `canRevokeVoice` - the holder of this capability can
- `canGrantVoice` - the holder of this capability can
- `canKnock` - the holder of this capability can
- `canAcceptKnock` - the holder of this capability can
- `canCreateSubgroup` - the holder of this capability can

These actions are subject to role constraints described below.

## Message Capabilities

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
- canSendDirectMessage
- canTargetMessage

The Hub can enforce whether a member can send a message. It can also withhold fanout of application messages to clients of a user. The other capabilities in this section can only be enforced by other clients.


## Asset Capabilities

- canUploadImage
- canUploadVideo
- canUploadAttachment
- canDownloadImage
- canDownloadVideo
- canDownloadAttachment
- canSendLink
- canSendLinkPreview
- canFollowLink

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
- canDestroyRoom
- MLS specific
  - update
  - reinit
  - PSK
  - external proposal
  - external commit


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
canRevokeVoice
canGrantVoice
canKnock
canAcceptKnock
canChangeUserRole
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
