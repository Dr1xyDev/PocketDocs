# 11 — AvailableCommandsPacket JSON schema

`AvailableCommandsPacket` (`0x4e`, see [09-player-input-and-ui.md](09-player-input-and-ui.md)) carries the command list the client uses to build autocompletion, the `/help` output and argument hints. In protocol 113 the whole list is **one JSON document** transmitted as a String, followed by one empty String:

| Field | Type |
|-------|------|
| Commands JSON | String |
| Unknown | String (always `""` in practice) |

The JSON in protocol 113 is an **object keyed by command name**, where each command exposes exactly one *version* entry:

```json
{
  "<command name>": {
    "versions": [ <command version> ]
  }
}
```

## Command version object

The schema of a single command version (verified against the template shipped with protocol-113-era implementations):

```json
{
  "aliases": [],
  "description": "insert_description_here",
  "overloads": {
    "default": {
      "input": {
        "parameters": [
          { "name": "args", "type": "rawtext", "optional": true }
        ]
      },
      "output": {}
    }
  },
  "permission": "any"
}
```

| Field | Type | Meaning |
|-------|------|---------|
| `aliases` | array of String | alternative names the client accepts/autocompletes for the command |
| `description` | String | shown in `/help` and the command suggestion popup |
| `overloads` | object | keyed by overload name (usually `"default"`); each value is an overload |
| `overload.input` | object | input definition |
| `overload.input.parameters` | array | the command's arguments in order |
| `overload.output` | object | empty on the wire in 1.1 |
| `permission` | String | display-only marker (`"any"` in the era's implementations); real permission checks happen server-side |

## Parameters

Each parameter object:

| Field | Type | Meaning |
|-------|------|---------|
| `name` | String | argument label shown to the user |
| `type` | String | argument type, below |
| `optional` | Bool | `true` = the client marks the argument as optional |

Type strings seen on the wire in the protocol-113 era:

| Type | Meaning |
|------|---------|
| `rawtext` | free-form text; consumes the rest of the line (the type used by the era's default template) |
| `strarg` | single string argument |
| `intarg` | integer argument |
| `floatarg` | float argument |
| `cmdarg` | sub-command selector for enums (e.g. `add` / `set` / `remove` choices) |

> `rawtext` is the type verified byte-for-byte in the era's command template; `strarg` / `intarg` / `floatarg` / `cmdarg` appear in server implementations of the same era and are accepted by 1.1.x clients, but no vanilla-published enumeration for 113 exists, so treat the four as advisory.

## Worked example

Two commands as they would appear in the packet's JSON string:

```json
{
  "time": {
    "versions": [
      {
        "aliases": [],
        "description": "Sets or queries the world time",
        "overloads": {
          "default": {
            "input": {
              "parameters": [
                { "name": "action", "type": "cmdarg", "optional": false },
                { "name": "amount", "type": "intarg", "optional": true }
              ]
            },
            "output": {}
          }
        },
        "permission": "any"
      }
    ]
  },
  "say": {
    "versions": [
      {
        "aliases": [],
        "description": "Broadcasts a message to all players",
        "overloads": {
          "default": {
            "input": {
              "parameters": [
                { "name": "message", "type": "rawtext", "optional": false }
              ]
            },
            "output": {}
          }
        },
        "permission": "any"
      }
    ]
  }
}
```

## Client behaviour

* The client matches the text after `/` against command **names and aliases** from this JSON; unmatched input is sent anyway as a `CommandStepPacket` (`0x4f`) and the server decides.
* Parameter `type`/`optional` drive the on-screen argument hints; they are cosmetic — the server receives whatever the player typed.
* Commands the client does not know (JSON sent before join, or commands hidden per-player) still execute when submitted; the JSON only affects client-side presentation.
