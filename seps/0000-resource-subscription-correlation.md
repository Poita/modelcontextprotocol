# SEP-0000: Resource Subscription Correlation

- **Status**: Draft
- **Type**: Standards Track
- **Created**: 2026-01-20
- **Author(s)**: Peter Argany (@pja)
- **Sponsor**: None (seeking sponsor)
- **PR**: https://github.com/modelcontextprotocol/specification/pull/0000

## Abstract

This SEP proposes adding a `subscribedUri` field to `ResourceUpdatedNotificationParams` to enable reliable correlation between resource update notifications and the subscriptions that triggered them. Currently, when a server sends a `notifications/resources/updated` notification, the `uri` field may refer to a sub-resource or related resource rather than the originally subscribed URI, making it difficult for clients to route updates to the correct subscription callback.

## Motivation

The MCP specification currently allows servers to send resource update notifications with URIs that differ from the originally subscribed URI. The schema documentation for `ResourceUpdatedNotificationParams.uri` explicitly states: "This might be a sub-resource of the one that the client actually subscribed to."

While this flexibility is valuable for hierarchical and query-based resource patterns, it creates a significant problem: **clients cannot reliably determine which subscription triggered a given update notification**.

### Example Scenarios

1. **Sub-resource updates**: A client subscribes to `file:///home` and receives an update for `file:///home/documents/report.txt`. The client has no definitive way to know this update relates to its `file:///home` subscription.

2. **Hierarchical resources**: A client subscribes to `slack://channel/123` and receives an update for `slack://channel/123/messages/456`. While prefix matching might work here, it's not guaranteed to be correct.

3. **Query-based subscriptions**: A client subscribes to `slack://messages/query?author=pja` and receives an update for `slack://messages/789` (a message matching the query). There is no structural relationship between these URIs that would allow correlation.

4. **Ambiguous overlapping subscriptions**: A client subscribes to both `file:///home` and `file:///home/subdir`. When it receives an update for `file:///home/subdir/file.txt`, it cannot determine which subscription (or both) triggered the notification.

Without a mechanism to correlate updates to subscriptions, clients must resort to heuristics (like prefix matching) that may fail in legitimate use cases, or maintain complex mapping logic that duplicates server-side knowledge about resource relationships.

## Specification

### Schema Changes

Add a required `subscribedUri` field to `ResourceUpdatedNotificationParams`:

```typescript
export interface ResourceUpdatedNotificationParams extends NotificationParams {
  /**
   * The URI of the resource that has been updated. This might be a sub-resource of
   * the one that the client actually subscribed to.
   *
   * @format uri
   */
  uri: string;

  /**
   * The URI that was originally passed to `resources/subscribe`. This allows clients
   * to reliably correlate update notifications with their subscriptions, even when
   * the updated resource URI differs from the subscribed URI.
   *
   * @format uri
   */
  subscribedUri: string;
}
```

### Behavioral Requirements

1. **subscribedUri MUST match exactly**: The `subscribedUri` field MUST contain the exact URI string that was provided in the original `resources/subscribe` request. Servers MUST NOT normalize, canonicalize, or otherwise transform this URI.

2. **uri MAY differ from subscribedUri**: The `uri` field MAY contain:
   - The same value as `subscribedUri` (when the subscribed resource itself changed)
   - A sub-resource URI (e.g., a file within a subscribed directory)
   - A related resource URI (e.g., a message matching a subscribed query)
   - Any other URI that the server determines is relevant to the subscription

3. **Relationship is implementation-defined**: The semantic relationship between `subscribedUri` and `uri` is determined by the server's resource model. The specification does not mandate any particular relationship (e.g., prefix matching).

4. **Multiple subscriptions require multiple notifications**: If a single resource change affects multiple active subscriptions, the server MUST send a separate `notifications/resources/updated` notification for each affected subscription, each with the appropriate `subscribedUri` value.

### Example Notifications

**Direct resource update** (subscribed resource itself changed):
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "file:///home/config.json",
    "subscribedUri": "file:///home/config.json"
  }
}
```

**Sub-resource update** (file within subscribed directory):
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "file:///home/documents/report.txt",
    "subscribedUri": "file:///home"
  }
}
```

**Query-based subscription** (resource matching subscribed query):
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "slack://messages/789",
    "subscribedUri": "slack://messages/query?author=pja"
  }
}
```

**Overlapping subscriptions** (same change triggers two notifications):
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "file:///home/subdir/file.txt",
    "subscribedUri": "file:///home"
  }
}
```
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "file:///home/subdir/file.txt",
    "subscribedUri": "file:///home/subdir"
  }
}
```

## Rationale

### Why a new field instead of reusing `uri`?

The existing `uri` field serves an important purpose: it tells the client *which specific resource changed*. This is valuable information that clients may need for:
- Refreshing only the changed resource rather than re-reading an entire directory
- Displaying accurate change notifications to users
- Maintaining caches with fine-grained invalidation

Overloading `uri` to always contain the subscribed URI would lose this granularity.

### Why make `subscribedUri` required?

Making the field required ensures:
1. Clients can rely on its presence without version checking
2. Server implementations are forced to track subscription URIs
3. The correlation problem is definitively solved for all compliant implementations

The backward compatibility impact is acceptable (see below).

### Alternatives considered

1. **Capability flag approach**: A capability flag like `supportsSubscriptionCorrelation` was considered but rejected. It adds complexity without benefit since:
   - Clients must handle both cases anyway during transition
   - The field's presence serves as implicit feature detection
   - Required fields are simpler to implement and test

2. **Metadata extension**: Using the `_meta` field was considered but rejected because:
   - This is core protocol functionality, not metadata
   - It would make the field optional by convention
   - It complicates schema validation

3. **Subscription ID approach**: Assigning server-generated subscription IDs was considered but rejected because:
   - It requires clients to maintain additional state
   - URIs already serve as unique identifiers for subscriptions
   - It doesn't provide additional value over `subscribedUri`

## Backward Compatibility

This change adds a new required field to an existing notification type. The compatibility impact is as follows:

### Older clients with newer servers

Older clients that do not expect `subscribedUri` will receive it but should ignore unknown fields per standard JSON-RPC practices. This is the default behavior for most JSON parsing libraries and MCP implementations.

### Newer clients with older servers

Newer clients expecting `subscribedUri` will not receive it from older servers. Clients SHOULD:
1. Check for the presence of `subscribedUri` in received notifications
2. When absent, fall back to heuristic matching (e.g., prefix matching or exact match)
3. Log warnings when heuristic matching is required, to encourage server upgrades

### Migration path

1. **Phase 1 (Immediate)**: Servers MAY begin sending `subscribedUri` in notifications
2. **Phase 2 (Next spec version)**: `subscribedUri` becomes a required field in the schema
3. **Phase 3 (Deprecation period)**: Clients warn when `subscribedUri` is absent
4. **Phase 4 (Future)**: Clients may require `subscribedUri` and fail gracefully without it

No capability negotiation is required. The field's presence serves as implicit feature detection.

## Security Implications

This change has no direct security implications. The `subscribedUri` field contains information that was already known to the client (it originated from the client's subscription request).

Implementations should ensure that:
- Servers only send `subscribedUri` values that correspond to actual subscriptions from the receiving client
- Servers do not leak subscription information between clients in multi-tenant scenarios

## Reference Implementation

A reference implementation will be provided in the TypeScript SDK that demonstrates:
1. Server-side tracking of subscription URIs
2. Including `subscribedUri` in all resource update notifications
3. Client-side callback routing based on `subscribedUri`
4. Fallback behavior for older servers

The implementation should be straightforward as servers must already track subscriptions to know when to send notifications.

## Open Questions

1. Should `subscribedUri` be optional during a transition period, or required immediately in the next spec version?

2. Should there be guidance on how servers should handle subscription URI normalization (e.g., trailing slashes, case sensitivity)?
