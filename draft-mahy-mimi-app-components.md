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
It contains a role_index, which is used to refer to the role elsewhere.
It also contains a role_name (a human readable text string name for the
role), and a role_description (another string, which can have zero length).

Each Role also can contain constraints on the minimum and maximum number of participants, and the minimum and maximum number of active participants.
If the minimum number is zero, there is no minimum number of participants for that particular role.
If there is no maximum number of participants for a particular role, that parameter is absent.

A party with a particular Role which has the `canAddParticipant` capability is authorized to add the new participant with any of the target roles in `add_participant_role_indexes`.
A party with a particular Role which has the `canChangeUserRole` capability is authorized to change the role of a participant from a role represented by `authorized_role_changes.from_role_index` to any of the `target_role_indexes` in the same element of `authorized_role_changes`.

>This design results in each participant only having a single role at a time, with a single list of capabilities and an explicit list of allowed role transitions. It makes the authorization process for a verifier consistent regardless of the complexity of the set of authorization rules.

Some examples are provided in {{role-examples}.

RoleData is the format of the `data` field inside the ComponentData struct for the Role-Based Access Control component in the `application_data` GroupContext extension.

~~~ tls
/* See MIMI Capability Types IANA registry */
uint16 CapablityType;

struct {
   int from_role_index;
   int target_role_indexes<V>;
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
  int add_participant_role_indexes<V>;
  SingleSourceRoleChangeTargets authorized_role_changes<V>;
} Role;

struct {
  Role roles<V>;
} RoleData;

RoleData RoleUpdate;
~~~

RoleUpdate (which has the same format as RoleData) is the format of the `update` field inside the ApplicationDataUpdate struct in an ApplicationDataUpdate Proposal for the Role-Based Access Control component.
If the contents of the `update` field are valid and if the proposer is authorized to generate such an update, the value of the `update` field completely replaces the value of the `data` field.

>Note that in the MIMI environment, changing the definitions of roles is anticipated to be very rare over the lifetime of a room (for example changing a room which has grown dramatically from cooperatively moderated to explicitly moderated).

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

When the rules in a Preauthorized users struct permit the requester multiple roles, the requesting client may choose any of those roles according to local policy.
That policy may simply select whichever role has the most capabilities or the more "desirable" capabilities according to its policy. Alternatively, the client could allow the end-user to select which (authorized) role to adopt in a particular room.

~~~ tls
struct {
  int target_role_index;
  /* preauth_domain consists of ASCII letters, digits, and hyphens */
  opaque preauth_domain<V>;
  /* the remaining fields are in the form of a URI */
  opaque preauth_workgroup<V>;
  opaque preauth_group<V>;
  opaque preauth_user<V>;
} PreAuthPerRoleList;

struct {
  /* MLS Credential Type of the "claim"  */
  CredentialType credential_type;
  /* the binary representation of an X.509 OID, a JWT claim name string, */
  /* or the CBOR representation of a CWT claim (an int or tstr) */
  opaque id<V>;
} ClaimId;

struct {
  ClaimId claim_id;
  opaque claim_value<V>;
} Claim;

struct {
  /* when all claims in the claimset are satisfied, claimset is satisfied */
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
canCreateJoinLink
canUseJoinLink
canUnBan
canKick
canRevokeVoice
canGrantVoice
canKnock
canAcceptKnock
canChangeUserRole
canCreateSubgroup
canTargetMessage
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
canSendDirectMessage
canTargetMessage
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
canChangeOwnAvatar
canChangeOwnPresence
canChangeOwnMood
canJoinCall
canSendAudio
canReceiveAudio
canSendVideo
canReceiveVideo
canShareScreen
canViewSharedScreen
canChangeOtherPolicyAttribute
canDestroyRoom
canSendMLSUpdateProposal
cansendMLSReinitProposal
canSendMLSPSKProposal
canSendMLSExternalProposal
canSendMLSExternalCommit
~~~

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
