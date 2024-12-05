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

TODO Abstract


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}




# Participant List


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


# Role-Based Access Control




RoleData is the format of the `data` field inside the ComponentData struct for the Role-Based Access Control component in the `application_data` GroupContext extension.

~~~ tls
/* See MIMI Capability Types IANA registry */
uint16 CapablityType;

struct {
  int role_index;
  opaque role_name<V>;
  opaque role_description<V>;
  CapabilityType role_capabilities<V>;
  int minimum_participants_constraint;
  optional int maximum_participants_constraint;
  int minimum_active_participants_constraint;
  optional int maximum_active_participants_constraint;
} Role;

struct {
  Role roles<V>;
} RoleData;

RoleData RoleUpdate;
~~~

RoleUpdate (which has the same format as RoleData) is the format of the `update` field inside the ApplicationDataUpdate struct in an ApplicationDataUpdate Proposal for the Role-Based Access Control component.
If the contents of the `update` field are valid and if the proposer is authorized to generate such an update, the value of the `update` field completely replaces the value of the `data` field.

# Preauthorized Participants


PreAuthData is the format of the `data` field inside the ComponentData struct for the Preauthorized Participants component in the `application_data` GroupContext extension.

~~~ tls
struct {
  Role target_role;
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

# Room Metadata

Room Metadata is data about a room which might be displayed as human-readable information about a room, such as the name of a room, its

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
  Uri room_uri;
  UTF8String room_name;
  /* an https URI resolving to an avatar image */
  Uri room_avatar;
  UTF8String room_subject;
  UTF8String room_mood;
} RoomMetaData;

RoomMetaData RoomMetaUpdate;
~~~

RoomMetaUpdate (which has the same format as RoomMetaData) is the format of the `update` field inside the ApplicationDataUpdate struct in an ApplicationDataUpdate Proposal for the Room Metadata component.
If the contents of the `update` field are valid and if the proposer is authorized to generate such an update, the value of the `update` field completely replaces the value of the `data` field.

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
canChangeRoomSubject
canChangeRoomAvatar
canChangeOwnName
canChangeOwnPresence
canChangeOwnMood
canChangeOwnAvatar
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
