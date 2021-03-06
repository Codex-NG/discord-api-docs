# Gateways

Gateways are Discords form of real-time communication over secure websockets. Clients will receive events and data over the gateway they are connected to and send data over the REST API. For information about connecting to a gateway see the [Connecting](#DOCS_GATEWAY/connecting) section.

## Get Gateway % GET /gateway

Returns an object with a single valid WSS URL. Clients **should** cache this value, and only call this endpoint to retrieve a new URL if they are unable to establish a Gateway connection to the cached URL.

###### Example Response

```json
{
	"url": "wss://gateway.discord.gg/"
}
```

## Gateway Protocol Versions

Out of Services versions are versions who's subset of changes compared to the most recent version have been completely removed from the Gateway. When connecting with these versions, the gateway may reject your connection entirely.

| Version | Out of Service |
|------------|----------------|
| 5 | no |
| 4 | no |


## Gateway OP Codes/Payloads

###### Gateway Payload Structure

| Field | Type | Description | Present |
|-------|------|-------------|-------------|
| op | integer | opcode for the payload | Always |
| d | mixed (object, integer) | event data | Always |
| s | integer | sequence number, used for resuming sessions and heartbeats | Only for OP 0 |
| t | string | the event name for this payload | Only for OP 0 |

###### Gateway OP Codes

| Code | Name | Description |
|--------|----------|-----------------|
| 0 | Dispatch | dispatches an event |
| 1 | Heartbeat | used for ping checking |
| 2 | Identify | used for client handshake |
| 3 | Status Update | used to update the client status |
| 4 | Voice State Update | used to join/move/leave voice channels |
| 5 | Voice Server Ping | used for voice ping checking |
| 6 | Resume | used to resume a closed connection |
| 7 | Reconnect | used to tell clients to reconnect to the gateway |
| 8 | Request Guild Members | used to request guild members |
| 9 | Invalid Session | used to notify client they have an invalid session id |
| 10 | Hello | sent immediately after connecting, contains heartbeat and server debug information |
| 11 | Heartback ACK | sent immediately following a client heartbeat that was received |

### Gateway Dispatch

Used by the gateway to notify the client of events.

###### Gateway Dispatch Example

```json
{
	"op": 0,
	"d": {},
	"s": 42,
	"t": "GATEWAY_EVENT_NAME"
}
```

### Gateway Heartbeat

Used to maintain an active gateway connection. Must be sent every `heartbeat_interval` milliseconds after the [ready](#DOCS_GATEWAY/ready) payload is received. Note that this interval already has room for error, and that client implementations do not need to send a heartbeat faster than what's specified. The inner `d` key must be set to the last seq (`s`) received by the client. If none has yet been received you should send `null` (you cannot send a heartbeat before authenticating, however).

>info
> It is worth noting that in the event of a service outage where you stay connected to the gateway, you should continue to heartbeat and receive ACKs. The gateway will eventually respond and issue a session once it is able to.

###### Gateway Heartbeat Example

```json
{
	"op": 1,
	"d": 251
}
```

### Gateway Heartbeat ACK

Used for the client to maintain an active gateway connection. Sent by the server after receiving a [Gateway Heartbeat](#DOCS_GATEWAY/gateway-heartbeat)

###### Gateway Heartbeat ACK Example

```json
{
	"op": 11
}
```

### Gateway Hello

Sent on connection to the websocket. Defines the heartbeat interval that the client should heartbeat to. 

###### Gateway Hello Structure

| Field | Type | Description |
|-------|------|-------------|
| heartbeat_interval | integer | the interval (in milliseconds) the client should heartbeat with |
| _trace | array of strings | used for debugging, array of servers connected to |

###### Gateway Hello Example

```json
{
	"heartbeat_interval": 45,
	"_trace": ["discord-gateway-prd-1-99"]
}
```

### Gateway Identify

Used to trigger the initial handshake with the gateway.

###### Gateway Identify Structure

| Field | Type | Description |
|-------|------|-------------|
| token | string | authentication token |
| properties | object | connection properties |
| compress | bool | whether this connection supports compression of the initial ready packet |
| large_threshold | integer | value between 50 and 250, total number of members where the gateway will stop sending offline members in the guild member list |
| shard | array of two integers (shard_id, num_shards) | used for [Guild Sharding](#DOCS_GATEWAY/sharding) |

###### Example Gateway Identify Example

```json
{
	"token": "my_token",
	"properties": {
		"$os": "linux",
		"$browser": "my_library_name",
		"$device": "my_library_name",
		"$referrer": "",
		"$referring_domain": ""
	},
	"compress": true,
	"large_threshold": 250,
	"shard": [1, 10]
}
```

### Gateway Reconnect		

Used to tell clients to reconnect to the gateway. Clients should immediately reconnect, and use the resume payload on the gateway.

### Gateway Request Guild Members

Used to request offline members for a guild. When initially connecting, the gateway will only send offline members if a guild has less than the `large_threshold` members (value in the [Gateway Identify](#DOCS_GATEWAY/gateway-identify)). If a client wishes to receive all members, they need to explicitly request them. The server will send a [Guild Members Chunk](#DOCS_GATEWAY/guild-members-chunk) event in response.

###### Gateway Request Guild Members Structure

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | id of the guild to get offline members for |
| query | string | string that username starts with, or an empty string to return all members |
| limit | integer | maximum number of members to send or 0 to request all members matched |

###### Gateway Request Guild Members Example

```json
{
	"guild_id": "41771983444115456",
	"query": "",
	"limit": 0
}
```

### Gateway Resume

Used to replay missed events when a disconnected client resumes.

###### Gateway Resume Structure

| Field | Type | Description |
|-------|------|-------------|
| token | string | session token |
| session_id | string | session id |
| seq | integer | last sequence number received |

###### Gateway Resume Example

```json
{
	"token": "randomstring",
	"session_id": "evenmorerandomstring",
	"seq": 1337
}
```

### Gateway Status Update

Sent by the client to indicate a presence or status update.

###### Gateway Status Update Structure

| Field | Type | Description |
|-------|------|-------------|
| idle_since | ?integer | unix time (in milliseconds) of when the client went idle, or null if the client is not idle |
| game | ?object | either null, or an object with one key "name", representing the name of the game being played |

###### Gateway Status Update Example

```json
{
	"idle_since": 91879201,
	"game": {
		"name": "Writing Docs FTW"
	}
}
```

### Gateway Voice State Update

Sent when a client wants to join, move, or disconnect from a voice channel.

###### Gateway Voice State Update Structure

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | id of the guild |
| channel_id | ?snowflake | id of the voice channel client wants to join (null if disconnecting) |
| self_mute | bool | is the client muted |
| self_deaf | bool | is the client deafened |

###### Gateway Voice State Update Example

```json
{
	"guild_id": "41771983423143937",
	"channel_id": "127121515262115840",
	"self_mute": false,
	"self_deaf": false
}
```



## Connecting

The first step to establishing a gateway connection is to request a gateway URL through the [Get Gateway](#DOCS_GATEWAY/get-gateway) API endpoint (if the client does not already have one cached). Using the "url" field from the response you can then create a new secure websocket connection that will be used for the duration of your gateway session. Once connected, the client will immediately receive an OP 10 [Hello](#DOCS_GATEWAY/gateway-hello) payload with the connection heartbeat interval. At this point the client should start sending OP 1 [heartbeat](#DOCS_GATEWAY/gateway-heartbeat) payloads every `heartbeat_interval` seconds. Next, the client sends an OP 2 [Identify](#DOCS_GATEWAY/gateway-identify) or OP 6 [Resume](#DOCS_GATEWAY/gateway-resume) payload. If your token is correct, the gateway will respond with a [Ready](#DOCS_GATEWAY/ready-event) payload.

###### Gateway URL Params

| Field | Type | Description |
|-------|------|-------------|
| v | integer | Gateway Version to use |
| encoding | string | 'json' or 'etf' |

### Resuming

When clients lose their connection to the gateway and are able to reconnect in a short period of time after, they can utilize a Gateway feature called "client resuming". Once reconnected to the gateway socket the client should send a [Gateway Resume](#DOCS_GATEWAY/gateway-resume) payload to the server. If successful, the gateway will respond by replaying all missed events to the client. Otherwise, the gateway will respond with an OP 9 (invalid session), in which case the client should send an OP 2 [Identify](#DOCS_GATEWAY/gateway-identify) payload to start a new connection. It is recommended that all Discord clients implement resume logic. The gateway can and will disconnect your websocket connection as it pleases without any warning. Implementing this feature will allow your client to reconnect seamlessly. Resuming is only supported when the amount of guilds in a given gateway connection is under 2,500. In order to resume as your bot grows, it's recommended that you [shard](#DOCS_GATEWAY/sharding) your gateway connection to reduce the number of guilds per gateway connection.

### Disconnections

If the gateway ever issues a disconnect to your client it will provide a close event code that you can use to properly handle the disconnection.

###### Gateway Close Event Codes

| Code | Description | Explanation |
|------|-------------|-------------|
| 4000 | unknown error | We're not sure what went wrong. Try reconnecting? |
| 4001 | unknown opcode | You sent an invalid [Gateway OP Code](#DOCS_GATEWAY/gateway-op-codes). Don't do that! |
| 4002 | decode error | You sent an invalid [payload](#DOCS_GATEWAY/sending-payloads) to us. Don't do that! |
| 4003 | not authenticated | You sent us a payload prior to [identifying](#DOCS_GATEWAY/gateway-identify). |
| 4004 | authentication failed | The account token sent with your [identify payload](#DOCS_GATEWAY/gateway-identify) is incorrect. |
| 4005 | already authenticated | You sent more than one identify payloads. Don't do that! |
| 4007 | invalid seq | The sequence sent when [resuming](#DOCS_GATEWAY/resuming) the session was invalid. Reconnect and start a new session. |
| 4008 | rate limited | Woah nelly! You're sending payloads to us too quickly. Slow it down! |
| 4009 | session timeout | Your session timed out. Reconnect and start a new one. |
| 4010 | invalid shard | You sent us an invalid [shard when identifying](#DOCS_GATEWAY/sharding). |

### ETF/JSON

When initially creating and handshaking connections to the Gateway, a user can chose whether they wish to communicate over plain-text JSON, or binary [ETF](http://erlang.org/doc/apps/erts/erl_ext_dist.html). Payloads to the gateway are limited to a maximum of 4096 bytes sent, going over this will cause a connection termination with error code 4002. Additionally, when using ETF, the client must not send compressed messages to the server.

### Rate Limiting

Unlike the HTTP API, the Gateway does not provide a method for forced back-off or cooldown but instead implement a hard limit on the number of messages sent over a period of time. Currently clients are allowed 120 events every 60 seconds, meaning you can send on average at a rate of up to 2 events per second. Clients who surpass this limit are immediately disconnected from the Gateway, and similarly to the HTTP API, repeat offenders will have their API access revoked. Clients are limited to one gateway connection per 5 seconds, if you hit this limit the Gateway will delay your connection until the cooldown has timed out.

>warn
> Clients may only update their game status 5 times per minute.

## Tracking State

Users who implement the Gateway API should keep in mind that Discord expects clients to track state locally and will only provide events for objects that are created/updated/deleted. A good example of state tracking is user status, when initially connecting to the gateway, the client receives information regarding the online status of members. However to keep this state updated a user must receive and track [Presence Update](#DOCS_GATEWAY/presence-update)'s. Generally clients should try to cache and track as much information locally to avoid excess API calls.

## Guild (Un)availability

When connecting to the gateway as a bot user, guilds that the bot is a part of start out as unavailable. Unavailable simply means that the gateway is either connecting, is unable to connect to a guild, or has disconnected from a guild. Don't fret however! When a guild goes unavailable the gateway will automatically attempt to reconnect on your behalf. As far as a client is concerned there is no distinction between a truly unavailable guild (meaning that the node that the guild is running on is unavailable) or a guild that the client is trying to connect to. As such, guilds only exist in two states: available or unavailable.

## Sharding

As bots grow and are added to an increasing number of guilds, some developers may find it necessary to break or split portions of their bots operations into separate logical processes. As such, Discord gateways implement a method of user-controlled guild-sharding which allows for splitting events across a number of gateway connections. Guild sharding is entirely user controlled, and requires no state-sharing between separate connections to operate.

To enable sharding on a connection, the user should send the `shard` array in the [identify](#DOCS_GATEWAY/gateway-identify) payload. The first item in this array should be the zero-based integer value of the current shard, while the second represents the total number of shards. DMs will only be sent to shard 0. To calculate what events will be sent to what shard, the following formula can be used:

```python
(guild_id >> 22) % num_shards == shard_id
```

As an example, if you wanted to split the connection between three shards, you'd use the following values for `shard` for each connection: `[0, 3]`, `[1, 3]`, and `[2, 3]`. Note that only the first shard (`[0, 3]`) would receive DMs.



## Payloads

### Sending Payloads

Packets sent from the client to the Gateway API are encapsulated within a [gateway payload object](#DOCS_GATEWAY/gateway-dispatch) and must have the proper OP code and data object set. The payload object can then be serialized in the format of choice, and sent over the websocket.

### Receiving Payloads

Receiving payloads with the Gateway API is slightly more complex than sending. When using the JSON encoding with compression enabled, the Gateway has the option of sending payloads as compressed JSON binaries using zlib, meaning your library _must_ detect and decompress these payloads before attempting to parse them. The gateway does not implement a shared compression context between messages sent.

### Event Names

Event names are in standard constant form, fully upper-cased and replacing all spaces with underscores. For instance, [Channel Create](#DOCS_GATEWAY/channel-create) would be `CHANNEL_CREATE` and [Voice State Update](#DOCS_GATEWAY/voice-state-update) would be `VOICE_STATE_UPDATE`.



## Events

### Ready

The ready event is dispatched when a client has completed the initial handshake with the gateway (for new sessions). The ready event can be the largest and most complex event the gateway will send, as it contains all the state required for a client to begin interacting with the rest of the platform.

###### Ready Event Fields

| Field | Type | Description |
|-------|------|-------------|
| v | integer | [gateway protocol version](#DOCS_GATEWAY/gateway-protocol-versions) |
| user | object | [user object](#DOCS_USER/user-object) (with email information) |
| private_channels | array | array of [DM channel](#DOCS_CHANNEL/dm-channel-object) objects |
| guilds | array | array of [Unavailable Guild](#DOCS_GUILD/unavailable-guild-object) objects |
| session_id | string | used for resuming connections |
| presences | array | list of friends' presences (not applicable to bots) |
| relationships | array | list of friends (not applicable to bots) |
| _trace | array of strings | used for debugging, array of servers connected to |

### Resumed

The resumed event is dispatched when a client has sent a [resume payload](#DOCS_GATEWAY/resuming) to the gateway (for resuming existing sessions).

###### Resumed Event Fields

| Field | Type | Description |
|-------|------|-------------|
| _trace | array of strings | used for debugging, array of servers connected to |

### Channel Create

Sent when a new channel is created, relevant to the current user. The inner payload is a [DM](#DOCS_CHANNEL/dm-channel-object) or [Guild](#DOCS_CHANNEL/guild-channel-object) channel object.

### Channel Update

Sent when a channel is updated. The inner payload is a [guild channel](#DOCS_CHANNEL/guild-channel-object) object.

### Channel Delete

Sent when a channel relevant to the current user is deleted. The inner payload is a [DM](#DOCS_CHANNEL/dm-channel-object) or [Guild](#DOCS_CHANNEL/guild-channel-object) channel object.

### Guild Create

This event can be sent in three different scenarios:

1. When a user is initially connecting, to lazily load and backfill information for all unavailable guilds sent in the [ready](#DOCS_GATEWAY/ready) event
2. When a Guild becomes available again to the client.
3. When the current user joins a new Guild.

The inner payload is a [guild](#DOCS_GUILD/guild-object) object, with an extra presences key containing an array of simple presence objects. The simple presence objects each have the same fields as a [Presence Update event](#DOCS_GATEWAY/presence-update) without a roles or guild_id key.

### Guild Update

Sent when a guild is updated. The inner payload is a [guild](#DOCS_GUILD/guild-object) object.

### Guild Delete

Sent when a guild becomes unavailable during a guild outage, or when the user leaves or is removed from a guild. See GUILD_CREATE for more information about how to handle this event.

###### Guild Delete Event Fields

| Field | Type | Description |
|-------|------|-------------|
| id | snowflake | id of the guild |
| unavailable | bool | whether the guild is unavailable, should always be true. if not set, this signifies that the user was removed from the guild |

### Guild Ban Add

Sent when a user is banned from a guild. The inner payload is a [user](#DOCS_USER/user-object) object, with an extra guild_id key.

### Guild Ban Remove

Sent when a user is unbanned from a guild. The inner payload is a [user](#DOCS_USER/user-object) object, with an extra guild_id key.

### Guild Emoji Update

Sent when a guilds emojis have been updated.

###### Guild Emoji Update Event Fields

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | id of the guild |
| emojis | array | array of [emojis](#DOCS_GUILD/emoji-object) |

### Guild Integrations Update

Sent when a guild integration is updated.

###### Guild Integrations Update Event Fields

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | id of the guild whose integrations where updated |

### Guild Member Add

Sent when a new user joins a guild. The inner payload is a [guild member](#DOCS_GUILD/guild-member-object) object with these extra fields:

###### Guild Member Add Extra Fields

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | id of the guild |

### Guild Member Remove

Sent when a user is removed from a guild (leave/kick/ban).

###### Guild Member Remove Event Fields

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | the id of the guild |
| user | a [user](#DOCS_USER/user-object) object | the user who was removed |

### Guild Member Update

Sent when a guild member is updated.

###### Guild Member Update Event Fields

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | the id of the guild |
| roles | array of [role](#DOCS_PERMISSIONS/role-object) objects | user roles |
| user | a [user](#DOCS_USER/user-object) object | the user |

### Guild Members Chunk

Sent in response to [Gateway Request Guild Members](#DOCS_GATEWAY/gateway-request-guild-members).

###### Guild Members Chunk Event Fields

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | the id of the guild |
| members | array of [guild members](#DOCS_GUILD/guild-member-object) | set of guild members |

### Guild Role Create

Sent when a guild role is created.

###### Guild Role Create Event Fields

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | the id of the guild |
| role | a [role](#DOCS_PERMISSIONS/role-object) object | the role created |

### Guild Role Update

Sent when a guild role is updated.

###### Guild Role Update Event Fields

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | the id of the guild |
| role | a [role](#DOCS_PERMISSIONS/role-object) object | the role created |

### Guild Role Delete

Sent when a guild role is deleted

###### Guild Role Delete Event Fields

| Field | Type | Description |
|-------|------|-------------|
| guild_id | snowflake | id of the guild |
| role_id | snowflake | id of the role |

### Message Create

Sent when a message is created. The inner payload is a [message](#DOCS_CHANNEL/message-object) object.

### Message Update

Sent when a message is updated. The inner payload is a [message](#DOCS_CHANNEL/message-object) object.

>warn
> Unlike creates, message updates may contain only a subset of the full message object payload (but will always contain an id and channel_id).

### Message Delete

Sent when a message is deleted.

###### Message Delete Event Fields

| Field | Type | Description |
|-------|------|-------------|
| id | snowflake | the id of the message |
| channel_id | snowflake | the id of the channel |

### Message Delete Bulk

Sent when multiple messages are deleted at once.

###### Message Delete Bulk Event Fields

| Field | Type | Description |
|-------|------|-------------|
| ids | array of snowflakes | the ids of the messages |
| channel_id | snowflake | the id of the channel |

### Presence Update

A user's presence is their current state on a guild. This event is sent when a user's presence is updated for a guild.

>warn
> The user object can be partial. The only field that it must contain is the "id" - every other field is optional, and won't be sent if they aren't updated.

###### Presence Update Event Fields

| Field | Type | Description |
|-------|------|-------------|
| user | [user](#DOCS_USER/user-object) object | the user presence is being updated for |
| roles | array of snowflakes | roles this user is in |
| game | object | null, or an object containing one key of "name" |
| nick | string | nickname of the user in the guild |
| guild_id | snowflake | id of the guild |
| status | string | either "idle", "online" or "offline" |

### Typing Start

Sent when a user starts typing in a channel.

###### Typing Start Event Fields

| Field | Type | Description |
|-------|------|-------------|
| channel_id | snowflake | id of the channel |
| user_id | snowflake | id of the user |
| timestamp | timestamp | when the user started typing |

### User Settings Update

Sent when the current user updates their settings. Inner payload is a [user settings](#DOCS_USER/user-settings-object) object.

### User Update

Sent when properties about the user change. Inner payload is a [user](#DOCS_USER/user-object) object.

### Voice State Update

Sent when someone joins/leaves/moves voice channels. Inner payload is a [voice state](#DOCS_VOICE/voice-state-object) object.

###### Example Voice State Update Event

```json
{
	"user_id": "104694319306248192",
	"session_id": "my_session_id"
}
```

### Voice Server Update

Sent when a guild's voice server is updated. This is sent when initially connection to voice, and when the current voice instance fails over to a new server.

###### Voice Server Update Event Fields

| Field | Type | Description |
|-------|------|-------------|
| token | string | voice connection token |
| guild_id | snowflake | the guild this voice server update is for |
| endpoint | string | the voice server host |

###### Example Voice Server Update Payload

```json
{
	"token": "my_token",
	"guild_id": "41771983423143937",
	"endpoint": "smart.loyal.discord.gg"
}
```
