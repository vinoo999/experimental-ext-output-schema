# A2A Protocol Extension: Output Schema (v1 Draft)

- **URI:** `https://a2a-protocol.org/extensions/output-schema/v1`
- **Type:** Profile Extension
- **Version:** 1.0.0 (draft)
- **Authors:** Vinay Ramesh (Bloomberg)

## 1. Abstract

LLMs can produce output in a structure the caller chooses, through JSON Schema constrained decoding
or prompt instructions. An agent that delegates to another agent over A2A cannot pass that request
along. A2A lets a client choose among declared output modes and send `acceptedOutputModes`, but a
media type names a format, not a shape.

This extension lets a client attach a schema to a `SendMessage` request. The schema describes the
shape of the result the client wants, in a declared schema language: JSON Schema, XSD, a natural
language description, or another language identified by URI or media type. The agent uses it as a
hint when producing its result.

The schema is a generation hint, not a contract. Validation is left to SDKs and implementations.
When a client supplies a schema that carries the structure it needs, agent harnesses MAY be able to
apply optimizations internally, for example by passing it to a model's structured output feature.

### 1.1 Definitions

- **Output schema request:** The object defined in [Section 4.2](#42-wire-format).
- **Request message:** The `Message` in the same `SendMessageRequest` as an output schema request.
- **Structured output Part:** The `Part` an agent produces in response to an output schema request.

## 2. Extension URI

- **URI:** `https://a2a-protocol.org/extensions/output-schema/v1`

The URI is an identifier. HTTP access to it is not expected. A breaking change requires a new URI
(Section 4.6.3 of the A2A specification).

## 3. Extension Capability Declaration

An agent that accepts output schema requests MUST declare the extension in
`AgentCard.capabilities.extensions`. The extension SHOULD NOT be declared with `required: true`,
because the request is an optional hint.

```json
{
  "uri": "https://a2a-protocol.org/extensions/output-schema/v1",
  "description": "Accepts a client-supplied schema for the shape of the result.",
  "required": false,
  "params": {
    "schemaTypes": [
      {
        "type": "json-schema",
        "dialects": [
          "https://json-schema.org/draft/2020-12/schema",
          "http://json-schema.org/draft-07/schema#"
        ]
      },
      { "type": "xsd", "dialects": ["1.0"] },
      { "type": "prompt" }
    ],
    "maxSchemaBytes": 65536,
    "remoteSchemas": false
  }
}
```

| Field                    | Type            | Required | Description |
| :----------------------- | :-------------- | :------- | :---------- |
| `schemaTypes`            | array of object | Yes      | Schema types the agent accepts. |
| `schemaTypes[].type`     | string          | Yes      | A schema type identifier (see below). |
| `schemaTypes[].dialects` | array of string | No       | Accepted dialects or versions. For `json-schema`, dialect URIs as used in `$schema`. If absent for `json-schema`, only `https://json-schema.org/draft/2020-12/schema` is accepted. |
| `maxSchemaBytes`         | integer         | No       | Largest accepted schema, in bytes of its UTF-8 serialization. |
| `remoteSchemas`          | boolean         | No       | `true` if the agent fetches `schema.url` and remote references inside a schema. Default `false`. |

Schema type identifiers:

| Identifier    | Schema language | `definition` form |
| :------------ | :-------------- | :---------------- |
| `json-schema` | JSON Schema [JSON Schema 2020-12](https://json-schema.org/draft/2020-12) | JSON object or boolean |
| `xsd`         | W3C XML Schema [XSD 1.1](https://www.w3.org/TR/xmlschema11-1/) | string holding the schema document |
| `prompt`      | Natural language description of the structure | string |

Any other schema type MUST be an absolute URI [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986) that identifies the schema language, or the
media type of the schema document. Agents SHOULD treat `application/schema+json` as `json-schema`.

## 4. Client Extension Request

### 4.1 Extension Activation

A client activates the extension by including the URI in the `A2A-Extensions` service parameter
(Section 3.2.6 of the A2A specification). An agent that supports the extension SHOULD echo the URI in
the `A2A-Extensions` service parameter of its response.

If the URI is not in `A2A-Extensions`, the agent SHOULD ignore any output schema request. An agent
that does not support the extension ignores the metadata, as for any unknown extension. Because the
extension is not required, an agent MUST NOT return `ExtensionSupportRequiredError` for it.

Clients SHOULD pick a schema type and dialect the agent declares, and SHOULD NOT send output schema
requests to agents that do not declare the extension, because a schema can reveal the client's data
model.

### 4.2 Wire Format

The output schema request is a JSON object at
`SendMessageRequest.metadata["https://a2a-protocol.org/extensions/output-schema/v1"]`.

| Field               | Type           | Required    | Description |
| :------------------ | :------------- | :---------- | :---------- |
| `schema.type`       | string         | Yes         | Schema type identifier ([Section 3](#3-extension-capability-declaration)). |
| `schema.dialect`    | string         | No          | Dialect or version. For `json-schema`, a dialect URI. A `$schema` keyword in the definition takes precedence, as in the [Structured Input extension](https://github.com/a2aproject/experimental-ext-structured-input). |
| `schema.definition` | any JSON value | Conditional | The inline schema. Exactly one of `definition` and `url` MUST be present. |
| `schema.url`        | string         | Conditional | An `https` URL for the schema. Only usable if the agent declares `remoteSchemas: true`. |
| `name`              | string         | No          | Short identifier for the shape. Agents SHOULD use it as `Artifact.name`. |
| `description`       | string         | No          | What the output is for. Agents MAY pass it to the model with the schema. |

A request carries at most one output schema request. A client that needs several outputs SHOULD
combine them into one schema.

The request is placed in `SendMessageRequest.metadata`, not `Message.metadata`, for two reasons.
First, it applies to one request, like `configuration.acceptedOutputModes` next to it. Second,
`Task.history` stores `Message` objects, so a schema in `Message.metadata` would be kept with the
Task and appear to apply on later turns. Nothing in `Task` retains `SendMessageRequest.metadata`.

### 4.3 Scope

An output schema request applies to the request it is sent with. Whether it continues to apply on
later turns of the same Task is up to the agent. A client that needs the same shape on a later turn
SHOULD send it again. A newer request takes precedence over any earlier one the agent kept. A new
Task never inherits an output schema request from another Task.

## 5. Agent Behavior

### 5.1 Applying the Schema

An agent that receives an output schema request it supports SHOULD use the schema to shape its
result. This specification does not prescribe how: through a model's structured output feature,
through instructions to the model, by mapping it to an internal workflow, or otherwise. The request
does not change what work the agent performs or what it may return.

The result SHOULD be a single structured output Part, in an `Artifact` for a Task or in the response
`Message`. JSON results SHOULD use the `data` field rather than JSON encoded in `text`. Other text
formats such as XML SHOULD use `text`. Other Parts (for example a short explanation) MAY be present.

Format negotiation is unchanged. `acceptedOutputModes`, the agent's declared output modes, and
`ContentTypeNotSupportedError` keep their core meaning.

### 5.2 Conformance Hint

An agent MAY add a hint to the structured output Part in
`Part.metadata["https://a2a-protocol.org/extensions/output-schema/v1"]`:

- `conformance`: `validated` (the agent validated the result against the schema as received, and it
  passed), `best-effort` (the schema guided generation without that validation), or `not-applied`.
  An agent MUST NOT report `validated` without performing that validation.
- `requestMessageId`: the `messageId` of the request message whose schema was applied. This tells
  the client whether the agent carried a schema forward from an earlier turn.

The hint is information, not proof. Its absence means only that the agent reported nothing. Clients
that need conformance SHOULD validate the result themselves.

### 5.3 Fallback

This extension adds no error types. An agent that cannot apply a request (unsupported schema type or
dialect, schema too large, remote schema not fetched) SHOULD process the message as if no request
had been sent. It MAY report `not-applied` in `Task.metadata` (or the response `Message.metadata`)
under the extension URI. An agent MAY reject a malformed request, such as one with both or neither of
`definition` and `url`, with a validation error per Section 3.3.2 of the A2A specification.

## 6. Example

The agent declares the extension as in [Section 3](#3-extension-capability-declaration). The client
asks for a JSON result with a given shape:

```http
POST /a2a/v1/message:send HTTP/1.1
Host: research.example.com
Content-Type: application/a2a+json
A2A-Extensions: https://a2a-protocol.org/extensions/output-schema/v1

{
  "message": {
    "messageId": "9f1c2e7a-0b4d-4a53-8f0e-2b1d7c3e5a10",
    "role": "ROLE_USER",
    "parts": [{ "text": "Build a short profile of ACME Corp from its latest annual filing." }]
  },
  "configuration": { "acceptedOutputModes": ["application/json", "text/plain"] },
  "metadata": {
    "https://a2a-protocol.org/extensions/output-schema/v1": {
      "schema": {
        "type": "json-schema",
        "dialect": "https://json-schema.org/draft/2020-12/schema",
        "definition": {
          "type": "object",
          "properties": {
            "ticker": { "type": "string" },
            "sector": { "type": "string" },
            "revenueUsd": { "type": "number" },
            "risks": { "type": "array", "items": { "type": "string" }, "maxItems": 5 }
          },
          "required": ["ticker", "sector", "revenueUsd", "risks"],
          "additionalProperties": false
        }
      },
      "name": "company_profile"
    }
  }
}
```

Response:

```http
HTTP/1.1 200 OK
Content-Type: application/a2a+json
A2A-Extensions: https://a2a-protocol.org/extensions/output-schema/v1

{
  "task": {
    "id": "3b8e6f02-5c1a-4d0e-9a7b-61f4c2d8e913",
    "contextId": "c4a1d9b2-7e3f-4b60-8d25-0f9e1a6c7b48",
    "status": { "state": "TASK_STATE_COMPLETED" },
    "artifacts": [
      {
        "artifactId": "a1",
        "name": "company_profile",
        "parts": [
          {
            "data": {
              "ticker": "ACME",
              "sector": "Industrials",
              "revenueUsd": 4210000000,
              "risks": ["Customer concentration", "Pending litigation"]
            },
            "mediaType": "application/json",
            "metadata": {
              "https://a2a-protocol.org/extensions/output-schema/v1": {
                "conformance": "validated",
                "requestMessageId": "9f1c2e7a-0b4d-4a53-8f0e-2b1d7c3e5a10"
              }
            }
          }
        ]
      }
    ]
  }
}
```

A `prompt` schema uses the same request shape. It cannot be validated, so an agent reports at most
`best-effort`:

```json
{
  "schema": {
    "type": "prompt",
    "definition": "A markdown table with columns Ticker, Company, Revenue growth (%), highest growth first, at most 10 rows, no other text."
  }
}
```

## 7. Interaction with Core A2A

- **Multi-turn:** Scope follows [Section 4.3](#43-scope). Status messages in
  `TASK_STATE_INPUT_REQUIRED` are not structured output.
- **Streaming:** Agents SHOULD NOT split a `data` Part across `TaskArtifactUpdateEvent` chunks,
  because A2A does not define how appended JSON values merge. A conformance hint, if present, SHOULD
  be on the chunk with `lastChunk: true`.
- **Extension-unaware peers:** Unaware clients send nothing and see no change. Unaware agents ignore
  the metadata. A client MAY also describe the format in a text Part for such agents.

## 8. Security and Privacy

- **Untrusted input:** The schema, including `description`, `title`, and `prompt` text, is client
  data. Agents MUST NOT give it the authority of system instructions, and SHOULD treat it as a
  possible prompt injection vector.
- **No authorization effect:** A schema can ask for fields the client has no right to see. Agents
  MUST apply the same access policy to structured output as to any other response.
- **Remote fetch:** Agents that declare `remoteSchemas: true` MUST fetch only `https` URLs and SHOULD
  apply SSRF protections (address checks on every redirect, timeouts, size limits) [OWASP SSRF guidance](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html).
  This covers remote `$ref` and `xs:import`/`xs:include`. Other agents MUST NOT dereference remote
  references.
- **Resource limits:** Agents SHOULD limit schema size, nesting, and `$ref` cycles, and SHOULD guard
  against regular expression backtracking in `pattern` keywords. Agents that parse XSD MUST disable
  external entity resolution.
- **Privacy:** A schema can reveal the client's data model. An agent that keeps a schema across turns
  SHOULD apply the same retention rules as for message content, and SHOULD NOT share it across
  clients or tasks.
