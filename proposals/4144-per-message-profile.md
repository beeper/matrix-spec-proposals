# MSC4144: Per-message profiles
Currently profiles in Matrix are defined by `m.room.member` state events, and
there is no easy way to have different profiles per message.

## Proposal
The proposed solution is a new field called `m.per_message_profile`, which
contains a displayname and/or avatar URL to override the default profile,
plus an ID to identify different profiles within the same Matrix user.

```json
{
  "msgtype": "m.text",
  "body": "Hello, World!",
  "m.per_message_profile": {
    "id": "meow",
    "displayname": "cat",
    "avatar_url": "mxc://maunium.net/hgXsKqlmRfpKvCZdUoWDkFQo"
  }
}
```

The `id` field is required and is an opaque string. Clients may use it to group
messages with the same ID like they would group messages from the same sender.
For example, bridges would likely set it to the immutable remote user ID.

### Encrypted avatars
Because the profile is inside the ciphertext in encrypted events, the entire
profile can be hidden the server, as long as the avatar is also encrypted.
Encrypted avatars are placed under `avatar_file` instead of `avatar_url`.
The `avatar_file` field has the same schema as the `file` field in
`m.room.message` events.

<details>
<summary>Encrypted avatar example</summary>

```json
{
  "msgtype": "m.text",
  "body": "Hello, World!",
  "m.per_message_profile": {
    "id": "meow",
    "displayname": "cat",
    "avatar_file": {
      "v": "v2",
      "key": {
        "alg": "A256CTR",
        "ext": true,
        "k": "8dXeBMBMthuXGY5zmUh9Mi0aqC1kndMZ4NCa-0RhELc",
        "key_ops": [
          "encrypt",
          "decrypt"
        ],
        "kty": "oct"
      },
      "iv": "L6zup2cR570AAAAAAAAAAA",
      "hashes": {
        "sha256": "/cTs+PajUcznbV3h1w5gh1AHnLjrKQVl2jU3xLCqoBI"
      },
      "url": "mxc://maunium.net/eKLhozQduElYSgBkWjtwSXoi"
    }
  }
}
```

</details>

### Behavior of omitted and empty fields
If the `displayname` field is omitted, null, or an empty string, the
displayname from the member event should be used instead. Setting an empty
displayname using a per-message profile is not supported, as there aren't any
clear use cases for it.

However, there are use cases for setting an empty avatar, so `avatar_url` being
an empty string should be treated as clearing the avatar and falling back to
the client's default blank avatar behavior (e.g. generating one based on the
displayname). If both `avatar_url` and `avatar_file` are omitted or null, the
avatar from the member event should be used instead.

### Extensible profiles
This MSC is not related to extensible profiles and does not attempt to
implement them. However, in case extensible profiles are implemented as
something that can be referenced (e.g. room IDs), the MSC adding them could
allow per-message profiles to specify which extensible profile is used.

## Use cases

### Bridging
Per-message profiles will allow making "lighter-weight" bridges that don't need
appservice access. Currently the only option for such bridges is to prepend the
displayname to the message, which is extremely ugly.

Such bridges would obviously have downsides, like not being able to start chats
via standard mechanisms, and not being able to see the member list on Matrix.
However, those may be acceptable compromises for non-puppeting bridges that
only operate in specific predetermined rooms.

### Feature-parity with other platforms
Other chat applications such as Slack and Discord have "webhooks" which allow
per-message profile overrides. This MSC effectively enables the same on Matrix.

### Roleplaying, plural users, etc
Some users want to be able to switch between profiles quickly, which would be
much easier using this MSC. Currently easiest way is to have multiple accounts,
which has other benefits, but is much more cumbersome to manage.

## Potential issues
Implementing encrypted avatars could cause difficulty for clients that assume
that avatars are always unencrypted mxc URIs.

## Alternatives
Per-message profiles could be transmitted more compactly by defining the profile
in a new state event and only referencing the state key in the message event.
However, that approach wouldn't enable encrypting per-message profiles.

## Security considerations

### Preventing impersonation
To prevent impersonation using per-message profiles, clients should somehow
indicate to the user that the message has a per-message profile with an easy
way to see the user's MXID or default profile. For example, a client could have
a small `via @user:example.com` text next to the per-message displayname.

## Unstable prefix
`com.beeper.per_message_profile` should be used instead of `m.per_message_profile`
until this MSC is accepted.