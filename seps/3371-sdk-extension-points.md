# SEP-3371: Consistent SDK Extension Points

- **Status**: Draft
- **Type**: Standards Track
- **Created**: 2026-09-18
- **Author(s)**: Sambhav Kothari (@sambhav)
- **Sponsor**: None
- **PR**: https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3371

## Abstract

This SEP gives Tier 1 MCP SDKs five public extension points: registration, capability negotiation, custom messages, middleware, and metadata access.

SDK maintainers provide these extension points. Extension authors and application developers implement the behavior in their own packages. Each SDK chooses APIs that suit its language.

## Motivation

[SEP-2133](./2133-extensions.md) defines extension identification and advertisement but leaves SDK integration open. Missing hooks mean an extension author may need changes in several SDKs before they can release a feature.

The distinction is:

- **MCP protocol extensions** define capabilities, methods, or behavior that clients and servers understand. They use namespaced extension identifiers.
- **SDK extension points** are the public APIs used to implement those features: capability registration, custom method registration, middleware, and filters.

Middleware and filters also support local behavior, such as metrics or request normalization, without implementing a protocol extension. Their implementations can live in application code or external packages.

Throughout this proposal, “extension” means a protocol extension. Registering one means enabling its local implementation in the client or server. This SEP does not introduce a separate plugin system.

## Goals

- Support independently maintained extensions on both clients and servers.
- Allow request, response, notification, and `_meta` mutation through public APIs.
- Check extension dependencies, core capability requirements, and registration conflicts before processing messages.
- Make middleware order predictable and inspectable.
- Test equivalent behavior across SDKs without requiring identical APIs.

## Non-goals

- Bundling every extension or filter into each SDK.
- A shared plugin ABI, package format, dependency resolver, or dynamic loader.
- Changes to the core wire schema or generated protocol types.
- Support guarantees for alternative core result types or new transport behavior.

## Proposal

Tier 1 SDKs **MUST** meet the requirements below through documented public APIs. Other SDKs **SHOULD** support them too. The keywords **MUST** and **SHOULD** follow [BCP 14](https://www.rfc-editor.org/info/bcp14).

The requirements apply to clients and servers, within the message directions and lifecycle allowed by the selected protocol version. SDKs can reuse existing APIs, builders, adapters, or middleware; no shared class hierarchy or function signature is required.

### 1. Registration and validation

- Enable extensions explicitly by identifier. Each can contribute capabilities, custom methods, and middleware.
- Reject duplicate identifiers. A failed registration must leave no usable partial configuration; rejecting the whole configuration is acceptable.
- Reject method names already handled by the SDK, application, or another extension, and names reserved by the core protocol. Switching between request and notification registration cannot bypass a conflict.
- Keep the original handler after a rejected registration if the configuration remains usable.
- Let applications and extension configuration code inspect registered identifiers before processing messages. A read-only view is sufficient.
- Registration can be limited to startup; runtime installation and removal are optional.
- External packages must be able to use these APIs without being built into the SDK or added to an SDK-owned allowlist.

#### Validate local requirements

An extension can require another local extension by identifier, or an enabled core capability:

- `requires = ["com.example/search"]` requires that extension to be registered.
- `requires_core = ["resources", "tools.listChanged"]` requires resource support and the tool-list change feature.

These names illustrate SDK configuration, not new wire fields. Core requirements use existing capability names and flags for the relevant endpoint and protocol version.

Before processing messages, the SDK:

1. Collects the full configuration, so dependencies may be registered later than their users.
2. Checks every extension’s requirements, including dependencies of dependencies.
3. Rejects missing extensions and missing, disabled, unknown, or inapplicable core capabilities. Errors name the extension and unmet requirement; failed configuration cannot remain partially usable.

A present object such as `resources: {}` satisfies a presence requirement. A required boolean feature must be `true`.

Requirements do not install packages, enable capabilities, or set middleware order. Packages and applications handle version compatibility and shared interfaces. These checks cover your own client or server. Checking what the other side supports is described below.

### 2. Capability negotiation

Capabilities tell a client or server which features the other side supports. For example, before calling a custom search method, a client checks whether the server advertises the search extension.

SDKs can reuse their existing negotiation APIs, such as TypeScript’s `registerCapabilities()` and `getServerCapabilities()`.

This SEP requires those APIs to be available to extension packages so they can:

- Declare their support and settings.
- Read the other side’s support and settings before using a feature that needs them, including core features such as resources.

Use the existing structure from [SEP-2133](./2133-extensions.md):

- **Identifier:** `{vendor-prefix}/{extension-name}`. Third parties use a reversed domain they control, such as `com.example/search`. The `io.modelcontextprotocol` prefix is reserved for official extensions.
- **Maps:** `ClientCapabilities.extensions` and `ServerCapabilities.extensions`, keyed by that identifier. Use the same identifier for registration and dependencies.
- **Settings:** the extension’s settings object. `{}` means support without settings; a missing entry means support has not been declared.

Preserve settings and unrelated entries. Reject conflicting local settings for the same identifier; applications can supply the final settings before registration. Each extension decides when both sides need support and what to do if the required support is missing or incompatible, such as using a fallback or returning an error.

Follow the selected protocol version’s existing lifecycle. For 2026-07-28:

1. Clients declare support on each request in `params._meta["io.modelcontextprotocol/clientCapabilities"].extensions`.
2. Servers declare support in the `result.capabilities.extensions` field of `server/discover`.
3. Server extension code reads client support from the current request, never an earlier one.

No new negotiation RPC or common version field is needed. Local dependencies and middleware order stay in SDK configuration.

### 3. Custom messages

- Support registering handlers and sending messages for custom requests and notifications.
- Let extensions supply their own parameter and result types, including schemas or codecs where the language needs them.
- Route custom methods through normal dispatch, request correlation, errors, applicable cancellation, and middleware.
- Respect the selected protocol version’s message-direction rules; custom methods do not permit otherwise-prohibited unsolicited messages.

Custom messages use the checked registration path in §1. Existing application APIs can continue to support explicit handler replacement.

### 4. Middleware and filters

Applications can register filters directly, and extensions can contribute middleware to the same chains. Implementations may live in application code or external packages; the SDK supplies registration and execution hooks.

SDKs can offer separate filters, around-call middleware, or both:

- **Request filters:** inspect or change a request before sending it or invoking its handler.
- **Response filters:** inspect or change a successful result before sending it or returning it to the application.
- **Notification filters:** inspect or change a notification before sending or dispatching it, with no response.
- **Around-call middleware:** wrap processing, observe errors, reject a request, or return a valid result without invoking the handler.

Support these behaviors for core and custom methods, on clients and servers, including reverse calls where the protocol permits them. A separate filter need not have a `next` callback; the SDK’s APIs together must provide the required behavior.

- Registering middleware does not advertise an extension, capability, or custom method. If middleware uses a feature that needs support from the other side, it checks that support as described above.
- Filters can mutate an object or return a replacement, while preserving the method’s allowed request and result shapes. Arguments, content, and `_meta` can change; alternative core result types are outside this SEP.
- Error observation is required; error transformation is optional and must preserve a valid error message.
- Document when hooks run relative to parsing, validation, handler execution, and serialization. Raw-message access is optional; the SDK retains control of request IDs and `jsonrpc`.

Observation-only middleware, such as local timing, need not change any message.

#### Composition and order

Let applications set the order explicitly through a list, registration calls, or nested functions. SDKs should use their language’s usual composition style and document how it determines execution order, including repeated registration calls.

- Combine application and extension middleware in one configurable order for each chain. Document any default placement of extension middleware and let applications change it.
- Let applications and extension configuration code inspect the final order before processing messages. An ordered configuration list or equivalent view is enough; no middleware identifier scheme is required.
- An extension may contribute several middleware functions, and applications can place other middleware between them. Apply each contribution once when an application supplies an explicit order.
- Keep extension dependencies separate. Requiring an extension does not decide where its middleware runs. Packages document any order their middleware needs; applications arrange it in code.
- If sending and receiving use separate chains, each can have its own order.

For an execution order `[A, B]`:

1. A receives the request first, then B, then the handler.
2. Results and errors return through B, then A. Each sees changes from the middleware it called.
3. If B returns early, the handler and any later middleware are skipped; A still sees the outcome.
4. Separate filters follow the same order, skipping hooks they do not implement. Notifications run forward with no response phase.

Keep request-specific changes isolated from concurrent calls. This SEP requires explicit composition and inspection, not automatic sorting, numeric priorities, or `before`/`after` declarations.

### 5. Metadata access and preservation

- Expose `_meta` through the same hooks as other payload fields, including nested protocol objects exposed by the SDK.
- Allow entries to be added, updated, or removed without changing generated core types.
- Preserve valid metadata, including unknown entries, through serialization and typed APIs unless application or extension code changes it.
- Make mutations visible to subsequent middleware and the handler, receiving client or server, or calling application. Preserve the protocol version’s rules for reserved fields.

Extensions choose their metadata keys, values, and meaning. The SDK adds no extension-specific layout, does not automatically copy request metadata into results, and does not treat extension payload metadata as a declaration of support.

## Conformance

Test the SDK hooks separately from individual extensions. An extension does not have to use every hook, and still needs its own tests.

Each SDK supplies small client and server fixtures:

1. Install two synthetic extensions and standalone middleware through public APIs. Include an extension that contributes multiple middleware functions.
2. Accept a generated method name such as `com.example/search_<suffix>` and a nonce before startup.
3. Register that method and echo the nonce.

Generate the suffix from the test seed. The fixture must dispatch the supplied method name rather than a hard-coded one, while failures remain reproducible.

### Wire-observable scenarios

| Scenario                  | Required observation                                                                                                                                                                                                |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Custom messages           | Requests round-trip nested parameters and a nonce with correct IDs; unknown methods return method-not-found. Notifications reach the handler exactly once and produce no response, verified through fixture state.  |
| Capabilities              | Both extension entries and an unrelated application entry arrive with settings intact; disabled extensions are absent.                                                                                              |
| Mutation and metadata     | Core and custom request arguments, results, and notifications can change before their recipient observes them. Metadata additions, updates, removals, and unknown keys survive typed APIs and codecs.               |
| Composition               | The trace is `A request, B request, handler, B result, A result`; reversing the list reverses nesting. Application callbacks can sit between contributions from one extension, with each contribution applied once. |
| Early returns and errors  | An inner early return skips later middleware and the handler, then unwinds through outer middleware. Inner errors are observed while unwinding and reach the caller correctly.                                      |
| Isolation and observation | Concurrent calls retain separate nonces, metadata, and results. Standalone timing middleware works with no extension registered and changes neither messages nor advertisements.                                    |
| Disabled baseline         | Disabling extensions and middleware removes their methods, advertisements, and mutations without changing core behavior.                                                                                            |

Run applicable scenarios for both client and server fixtures, on each transport required for the SDK's tier. Version-specific fixtures follow that version's lifecycle and permitted message directions. Core messages retain core-schema validation; custom methods use the fixture's declared schemas and normal JSON-RPC envelope checks, rather than requiring membership in the core method union.

### SDK-local tests

Wire tests cannot establish how an SDK registers or runs extensions. Each SDK **MUST** also test:

- **Registration:** reject duplicate extension IDs, duplicate methods, and core method claims. Preserve existing handlers and leave no usable partial registration on failure.
- **Dependencies:** validate the completed configuration, including later registrations and dependencies of dependencies. Report missing IDs, disabled core flags, and invalid endpoint/version requirements. Distinguish an absent capability from `{}` and a disabled flag from `true`.
- **Ordering:** execution follows the application’s composition, including repeated registration calls and separate sending/receiving chains. Changing the configured order changes request order and reverses result order. Extension dependencies neither reorder middleware nor enable capabilities.
- **Inspection:** expose registered extensions and each chain’s final middleware order. Support several middleware functions from one extension with application middleware between them, without installing contributions twice.
- **Capabilities:** preserve independent entries and settings; let clients read server support and servers read client support from the current request. Extension payload metadata and local requirements do not establish support on the other side.
- **Public APIs:** external packages can register filters and mutate requests, results, notifications, and metadata without an SDK fork or extension registration. Separate filters preserve the same ordering and error/early-return behavior as the overall middleware API, skipping missing phase hooks.

The shared suite defines expected outcomes. SDKs use their native test frameworks and review fixture source to confirm that it uses public APIs.

## Rationale

- Requiring every SDK to implement each extension would keep SDK releases in the extension author’s release cycle.
- A shared plugin interface would constrain language-specific APIs without adding wire interoperability.
- Silent handler replacement makes behavior depend on installation order. Rejecting conflicts makes mistakes visible; filters provide an explicit way to wrap existing behavior.
- Alternative core result types need a separate decision. In particular, mutating a `CallToolResult` does not make a typed client accept a task handle in its place.

Explicit ordering is already familiar across languages: [Django](https://docs.djangoproject.com/en/5.2/topics/http/middleware/#middleware-order-and-layering) uses a list; [Koa](https://koajs.com/#cascading) and [ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0) use registration order; [Go](https://github.com/modelcontextprotocol/go-sdk/blob/3b917b466cc540079b82bcd0fecdc46a2ba13644/mcp/server.go) and [Tower](https://docs.rs/tower/0.5.3/tower/struct.ServiceBuilder.html#order) compose wrappers. These establish the composition pattern; MCP hooks still operate on protocol messages.

SDKs must document details such as repeated registration: Go's `AddReceivingMiddleware(a, b)` produces `a(b(handler))`, but separate calls wrap the current handler and therefore give a different order. The requirement is predictable composition and inspection, not identical syntax.

## Backward compatibility and adoption

- This SEP adds no core wire fields or methods. Applications that enable neither extensions nor middleware keep their existing behavior.
- SDKs can add checked extension registration while preserving existing application-level replacement APIs.
- Individual extensions remain optional under the [SDK tiering requirements](https://modelcontextprotocol.io/community/sdk-tiers). This SEP makes the hooks a separate Tier 1 requirement.
- On adoption, update the current SDK guidance and tiering documentation to distinguish optional extension implementations from required extension APIs. Preserve finalized SEP-2133 as a historical record.

Adoption steps:

1. Build the shared fixtures and SDK-local tests.
2. Have the SDK and Conformance working groups agree a migration window and adoption milestone.
3. Apply the new tier checks after that window, without changing frozen protocol-version conformance sets.

The adoption date remains open.

## Security considerations

- Extensions run as trusted application code and can read or change protocol data.
- Explicit opt-in, conflict checks, and visible order make behavior easier to review; they do not isolate extensions.
- Preserve authorization and protocol validation checks.
- Avoid exposing credentials in `_meta` and keep request-specific state isolated.
- Protocol middleware does not replace HTTP authentication.

## Open questions

- What migration milestone should make these checks mandatory for Tier 1?
- Should result-type extensibility be a follow-up requirement, with Tasks as its reference case?
- What is the smallest common reporting format for SDK-local contract-test results alongside shared wire conformance results?

## Appendix A: Checklist and worked examples

The five areas below match the proposal. Dependencies and collision checks belong to registration; request, result, notification, error, and ordering behavior belong to middleware. Each area may use one or more APIs suited to the SDK’s language.

| SDK extension point | Implementation checklist                                                                                                                                    | Example                                                                                                                          |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Registration        | Enable and inspect extensions; validate extension/core requirements; reject duplicate IDs and method claims without leaving usable partial configuration.   | [Enable search](#a1-enable-and-call-catalog-search)                                                                              |
| Capabilities        | Advertise settings, reject conflicting local settings, and read the other side's current support.                                                           | [Enable search](#a1-enable-and-call-catalog-search)                                                                              |
| Custom messages     | Register and send typed requests and notifications through normal dispatch, with correlation, errors, and cancellation where applicable.                    | [Search request](#a1-enable-and-call-catalog-search), [notification](#a3-record-a-search-notice-without-a-response)              |
| Middleware          | Register independently or through extensions; mutate requests/results/notifications, observe errors, and return early; compose and inspect execution order. | [Process search](#a2-normalize-validate-and-label-search-results), [notification](#a3-record-a-search-notice-without-a-response) |
| Metadata            | Add, update, remove, and preserve `_meta`, including unknown keys and metadata on nested protocol objects.                                                  | [Process search](#a2-normalize-validate-and-label-search-results)                                                                |

The examples follow a catalog service that searches invoices. An optional search package adds a custom method alongside an existing `search` tool. A formatting package can normalize queries and label results. Both entry points search the same catalog and, for this example, return text content blocks. A real custom method can define a different result schema.

Code is Python-like pseudocode, not a proposed SDK interface. Each example configures a fresh server before it starts; shared names and callbacks are reused. Application tool/resource handlers, connection setup, and schema declarations are omitted. The extension supplies its own parameter/result schemas; the SDK handles envelopes and request IDs. The pseudocode omits version-specific common result fields, which an implementation must include through its SDK’s result types or constructors. Callbacks receive request-local objects.

### A1. Enable and call catalog search

The application must opt into the search package before its custom method is available. The package requires local resource support because callers can read the documents it finds. The optional formatter depends on search. These checks catch incomplete server configuration before the first request.

```python
SEARCH = "com.example/search"
FORMAT = "com.example/search-format"

async def search_catalog(params):
    # Fixture for query="invoices", limit=2; real code queries and limits results.
    return {"content": [{"type": "text", "text": "2 invoices"}]}

search = Extension(SEARCH, requires_core=["resources"])
search.capability({"maxResults": 20})
search.request(SEARCH, handler=search_catalog)
formatting = Extension(FORMAT, requires=[SEARCH])

server = Server(
    extensions=[formatting, search],
    capabilities={"resources": {}},
)
assert server.extension_ids() == [FORMAT, SEARCH]
```

The SDK validates the completed configuration, so formatting may be listed before search. Removing search leaves an unmet extension dependency; removing resource support leaves an unmet core requirement. Neither requirement installs a package, enables a capability, or sets middleware order. A boolean requirement such as `tools.listChanged` needs `true`, whereas `resources: {}` satisfies a presence requirement.

The same registration path rejects a second `SEARCH` extension, a second handler for `SEARCH` (even a notification handler), or an extension claiming the core `tools/call` method. A failure cannot silently replace the existing handler or leave a partially installed extension usable.

The client checks what this server actually offers before calling it:

```python
info = await client.discover()
settings = info.capabilities.extensions.get(SEARCH)
if settings is None:
    use_core_search_tool()  # The application's existing fallback.
else:
    result = await client.request(SEARCH, {
        "query": "invoices", "limit": min(2, settings.get("maxResults", 20)),
    })
```

`search.capability(...)` puts the settings in the server's discovery result. The client uses them to stay within the advertised limit. Missing support selects the fallback; an empty settings object still declares support and uses this example's default limit of 20. Conflicting local settings must be resolved before registration, without overwriting unrelated capability entries.

This method only needs server support. If a feature also requires client support, the client registers its own capability entry. On 2026-07-28 the SDK sends it in each request's `params._meta["io.modelcontextprotocol/clientCapabilities"].extensions`; server code reads `request.client_capabilities.extensions` on that request. Local dependency checks cannot establish what the other side supports.

The SDK correlates the custom result with the request just as it does for a core method. In this example the result contains a text block saying `2 invoices`. Without a registered handler, the call instead receives the normal method-not-found error. No core method union needs editing.

### A2. Normalize, validate, and label search results

Now the application wants both entry points—the custom search method and the existing `search` tool—to trim whitespace, reject empty queries, and label output with its source. Putting this behavior in middleware lets a package supply it once instead of modifying each handler. Application timing can sit between the package's callbacks.

First, a helper finds the query in either request shape. Normalization runs before validation so a query containing only spaces cannot pass as nonempty:

```python
def search_args(request):
    if request.method == SEARCH:
        return request.params
    if request.method == "tools/call" and request.params["name"] == "search":
        return request.params["arguments"]
    return None

async def normalize(request, next):
    args = search_args(request)
    if args is not None:
        args["query"] = args["query"].strip()
    return await next(request)

async def validate(request, next):
    args = search_args(request)
    if args is not None:
        if not args["query"]:
            raise RpcError(-32602, "query must not be empty")
        if args.get("limit") == 0:
            # The sample schema permits zero: no catalog lookup is needed.
            return {"content": [{"type": "text", "text": "0 matches"}]}
    return await next(request)
```

Validation can reject a call or return a valid result without invoking the handler. Both outcomes still pass back through middleware that has already run. The next two callbacks use that return path: one labels successful results; the other records timing and errors without changing messages.

```python
async def label_results(request, next):
    result = await next(request)
    if search_args(request) is not None:
        for item in result["content"]:
            if item["type"] == "text":
                item["text"] += " (source: catalog)"
        meta = result.setdefault("_meta", {})
        own = meta.setdefault(FORMAT, {})
        own["source"] = "catalog"  # Add or update this package's field.
        own.pop("debug", None)     # Remove its internal detail, if present.
    return result

async def observe(request, next):
    started = clock.now()
    try:
        return await next(request)
    except RpcError as error:
        metrics.increment("rpc_errors", code=error.code)
        raise
    finally:
        metrics.record_duration(request.method, clock.now() - started)

formatting = Extension(
    FORMAT, requires=[SEARCH], middleware=[normalize, label_results],
)
chain = [normalize, observe, label_results, validate]
server = Server(
    extensions=[search, formatting], capabilities={"resources": {}},
    middleware=chain,
)
assert server.middleware_order() == chain
```

The list supplies the complete order; the SDK does not append the formatter's callbacks a second time. For `query: "  invoices  "`, `limit: 2`, normalization removes the spaces, validation passes, and the handler returns `2 invoices`. On the way back, `label_results` adds the source text and metadata; `observe` records the duration. The caller receives `2 invoices (source: catalog)`.

For a whitespace-only query, validation raises `-32602` before the handler runs. `label_results` does not process a successful result, but `observe` still records the error and duration. For `limit: 0`, validation returns `0 matches` immediately; the outer callbacks still label it and record timing. Reversing normalization and validation would incorrectly allow a whitespace-only query through the nonempty check.

The metadata edit changes only the formatter's fields. Other keys survive serialization and typed APIs; the same rule applies to metadata on individual content blocks. A package can remove its entire entry with `meta.pop(FORMAT, None)`, or set request metadata through `request.params.setdefault("_meta", {})`. Request metadata is not automatically copied to results and does not advertise capability support. Clients can ignore this source annotation, so this example requires no extra negotiation.

An application can also install `Server(middleware=[observe])` directly, with no extension registered. SDKs may expose separate request/result filters instead of around-call callbacks, provided their APIs together support the same behavior. Client-side hooks run before sending and before returning to the application; sending and receiving chains can have separate explicit orders.

### A3. Record a search notice without a response

Suppose the client records that a user selected a search result, and needs no value back. A custom notification avoids an unnecessary request/response exchange. A notification filter normalizes the query before the notice reaches the handler; it has no result phase to wrap.

This example uses a legacy session whose protocol and transport permit client-to-server custom notifications. It does not describe a client notification POST on the 2026-07-28 HTTP path. Custom registration never overrides the selected version's message-direction rules.

```python
NOTICE = "com.example/search-selected"
selected_queries = []

async def record_selection(params):
    selected_queries.append(params["query"])

def normalize_notice(notification):
    if notification.method == NOTICE:
        notification.params["query"] = notification.params["query"].strip()
    return notification

# Use a fresh search configuration and register before starting the server.
search.notification(NOTICE, handler=record_selection)
server = Server(
    extensions=[search], capabilities={"resources": {}},
    middleware=[Filters(notification=normalize_notice)],
)
await client.notify(NOTICE, {"query": "  invoices  "})
```

The wire message has no request ID. After dispatch, the handler has recorded `invoices` once; no JSON-RPC response is sent. Sending the notification does not confirm that it was handled—use a request if acknowledgment matters. An outbound filter could normalize before transmission instead, and permitted core notifications use the same filtering surface.

## Appendix B: Extension requirements

The columns match the five extension points in the proposal. This table asks which APIs an extension uses; the SDK table below asks whether those APIs are available.

**Legend:** ✅ used by the described integration; ◯ optional implementation choice; — not needed for that integration; ⚠️ useful but insufficient for the full behavior; ? design not concrete enough to assess. A check does not claim a completed implementation.

Registration applies when installing a package through this proposal, not as an extra wire requirement. Apps and Skills also require local resource support. None of the reviewed designs requires another named local extension. Experimental capability declarations may need adaptation to `extensions`.

| Extension                                                                                                                                                                                             | Registration | Capabilities | Custom messages | Middleware | Metadata |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :----------: | :----------: | :-------------: | :--------: | :------: |
| [Apps](https://github.com/modelcontextprotocol/ext-apps/blob/6d9bdc7babf275b759225aa722cbf5510c4c6021/specification/draft/apps.mdx)                                                                   |      ✅      |      ✅      |        —        |     ◯      |    ✅    |
| [Skills](https://github.com/modelcontextprotocol/ext-skills/blob/41e7c66db2510a3e98d9614eb1998f6b970006d7/specification/stable/skills.mdx)                                                            |      ✅      |      ✅      |       ✅        |     —      |    ✅    |
| [Tasks](https://github.com/modelcontextprotocol/ext-tasks/blob/9263312d11a682ac83f83fe84794d4627efd22f5/specification/draft/tasks.md)                                                                 |      ✅      |      ✅      |       ✅        |     ⚠️     |    ✅    |
| [Interceptors (experimental)](https://github.com/modelcontextprotocol/experimental-ext-interceptors/blob/b60459844cc95f2170297ebe1c84b7de8b752953/docs/sep.md)                                        |      ✅      |      ✅      |       ✅        |     ✅     |    ◯     |
| [Variants (experimental)](https://github.com/modelcontextprotocol/experimental-ext-variants/blob/cfc05d6f5eb8829f9896d44a6d47360bd15c3b5c/go/sdk/variants/server.go)                                  |      ✅      |      ✅      |        —        |     ⚠️     |    ◯     |
| [Triggers/events (experimental)](https://github.com/modelcontextprotocol/experimental-ext-triggers-events/blob/6682596d65eec778fe0b8b1f43b4e89d2fe2c546/docs/design-sketch-proposal.md)               |      ✅      |      ✅      |       ✅        |     ◯      |    ◯     |
| [Trust annotations (experimental)](https://github.com/modelcontextprotocol/experimental-ext-tool-annotations/blob/fecace78a9552f70ba735d750fc3c4b190e20429/specification/draft/trust-annotations.mdx) |      ✅      |      —       |        —        |     ◯      |    ✅    |
| [Action metadata (experimental)](https://github.com/modelcontextprotocol/experimental-ext-tool-annotations/blob/fecace78a9552f70ba735d750fc3c4b190e20429/specification/draft/action-metadata.mdx)     |      ✅      |      —       |        —        |     ◯      |    —     |
| [Grouping (exploratory)](https://github.com/modelcontextprotocol/experimental-ext-grouping/blob/2505387604eb144fb0d095de592dc4733e33f33b/README.md)                                                   |      ?       |      ?       |        ?        |     ?      |    ?     |

**Qualifications** (linked revisions reviewed on 2026-09-18):

- **Apps:** covers the MCP server connection. The separate app–host bridge has its own messages and lifecycle; UI hosting remains host/package work.
- **Skills:** adds `skills/list`, `skills/get`, and optional `resources/directory/read`; core resource handlers serve content. This revision needs no custom notifications.
- **Tasks ⚠️:** custom task requests and status notifications fit. Returning a task handle instead of the normal `tools/call` result needs alternative result-type support beyond middleware. Storage and execution remain package work.
- **Interceptors:** custom requests serve discovery and invocation; middleware applies interceptors to MCP calls. Agent lifecycle events need host hooks, and remote interceptor ordering is separate from local middleware order.
- **Variants ⚠️:** the linked proxy wraps requests/results and redirects notifications. Some routing needs session access beyond payload mutation, so these hooks do not guarantee every proxy topology.
- **Triggers/events:** the message types depend on delivery mode. Webhooks, long-lived streams, and timeout/concurrency exceptions may need additional APIs.
- **Trust annotations:** handlers can attach result/content-block metadata directly; result middleware is optional. The draft needs no capability negotiation.
- **Action metadata ⚠️:** adds fields to `Tool.annotations`, not `_meta`. These require typed-field preservation outside this proposal; optional result middleware does not solve that gap. Changes use core list-change behavior.
- **Grouping:** no sufficiently complete wire contract to assess.

Sending a notification does not itself require a notification filter. Likewise, attaching metadata in a handler does not require result middleware. Authorization extensions and Server Card need transport integration and remain outside the minimum contract.

## Appendix C: SDK support

The same five columns now show SDK support. This source review covers all ten official SDKs at the pinned revisions in the numbered notes, reviewed on 2026-09-18. Tiers follow the [official SDK listing](https://modelcontextprotocol.io/docs/2026-07-28/sdk); only Tier 1 adoption is required.

**Legend:** ✅ a usable public API exists for the scope shown; ❌ an observed gap; ? not established by this review. Public low-level APIs and straightforward wrappers count. A provisional API or a missing safeguard does not make an existing hook unavailable. These are source findings, not a claim that an SDK passes every proposed requirement.

| SDK / tier (source note) | Registration | Capabilities | Custom messages |     Middleware      | Metadata |
| ------------------------ | :----------: | :----------: | :-------------: | :-----------------: | :------: |
| TypeScript — Tier 1 (1)  |      ?       |      ✅      |       ✅        |          ?          |    ✅    |
| Python — Tier 1 (2)      |      ✅      |      ✅      |       ✅        |      ✅ server      |    ✅    |
| C# — Tier 1 (3)          |      ?       |      ✅      |       ✅        |      ✅ server      |    ✅    |
| Go — Tier 1 (4)          |      ?       |      ✅      |  ✅ requests¹   |         ✅          |    ✅    |
| Rust — Tier 1 (5)        |      ?       |      ✅      |       ✅        |     ✅ wrappers     |    ✅    |
| Java — Tier 2 (6)        |      ?       |      ❌      | ✅ session API  | ✅ handler wrappers |    ✅    |
| Ruby — Tier 2 (7)        |      ?       |      ✅      |  ✅ requests¹   |          ?          |    ✅    |
| PHP — Tier 3 (8)         |      ✅      |      ✅      |       ✅        |          ?          |    ✅    |
| Kotlin — Tier 3 (9)      |      ?       |      ✅      |       ✅        |          ?          |    ✅    |
| Swift — Tier 3 (10)      |      ?       |      ❌      |       ✅        |          ?          |    ✅    |

¹ Custom requests are established; complete custom-notification registration and dispatch coverage remains unverified. This does not make the request API partial.

The table separates **available building blocks** from **work needed to satisfy this SEP**:

- **Registration** means an extension-ID registration API, not merely method registration. Python and PHP provide one. Handler replacement in other SDKs is a collision-check concern; it is not evidence for or against an extension registry. Local dependencies, inspection, and failure rollback still need separate verification.
- **Capabilities** means extension-map support. Settings-conflict handling and version-specific negotiation still need verification.
- **Custom messages** means usable request/notification APIs in the described scope. Python's notification collision rules and C#'s experimental designation belong in the notes; they do not erase the underlying APIs. Java's public session constructors and send methods count even though high-level handler preparation is private.
- **Middleware** includes public composition points that an external package can wrap. Python and C# have documented server hooks; Go has sending/receiving middleware; Rust exposes the `Service` interface. Java's public session handler maps allow an application to supply decorated handlers; this is an integration through the low-level session API, not an established high-level middleware registry. Full client/server coverage and final-order inspection remain separate checks. HTTP-only hooks do not establish protocol middleware.
- **Metadata** means public `_meta` access. Preservation through every typed API and nested object still needs testing.

### Source notes

1. **TypeScript**

   - [Custom method support](https://github.com/modelcontextprotocol/typescript-sdk/blob/60321700871029401a2e3bed8fdf4f02c9ec3331/docs/advanced/custom-methods.md) supports requests, notifications, and capabilities, but replaces existing handlers.
   - [Documented client middleware](https://github.com/modelcontextprotocol/typescript-sdk/blob/60321700871029401a2e3bed8fdf4f02c9ec3331/docs/clients/middleware.md) wraps HTTP `fetch`. Verify protocol-level mutation hooks separately and add a checked registration path.

2. **Python**

   - [The extension API](https://github.com/modelcontextprotocol/python-sdk/blob/6affe5c0d3588fd1705713b3703dc68015cfe3eb/docs/advanced/extensions.md) provides identifiers, settings, and custom request bindings with conflict checks. Client notification bindings observe vendor messages; a core-shadowing notification binding is warned about rather than rejected, so stricter collision rejection still needs work. The request and client-notification APIs themselves are available.
   - [General server middleware](https://github.com/modelcontextprotocol/python-sdk/blob/6affe5c0d3588fd1705713b3703dc68015cfe3eb/docs/advanced/middleware.md) is provisional and server-side. Its public ordered list covers inbound request/result mutation, notifications that reach dispatch, errors, and early returns. Client coverage and mixed extension contribution placement still need verification; tool-call interception alone is insufficient.

3. **C#**

   - [Custom request registration](https://github.com/modelcontextprotocol/csharp-sdk/blob/324ccd83c357acf611e1cf3a6935a945ee600442/src/ModelContextProtocol.Core/Server/McpServerOptions.cs) is experimental and can override built-ins. Session APIs also send custom messages and handle notifications.
   - [Server message and request filters](https://github.com/modelcontextprotocol/csharp-sdk/blob/324ccd83c357acf611e1cf3a6935a945ee600442/docs/concepts/filters.md) cover the server, including notification processing and early returns. Registration order is documented, but full order inspection and equivalent client hooks remain unverified. Add guarded method registration.

4. **Go**

   - [Server custom-method registration](https://github.com/modelcontextprotocol/go-sdk/blob/3b917b466cc540079b82bcd0fecdc46a2ba13644/mcp/server.go) rejects core names but replaces duplicate custom handlers.
   - [Sending/receiving middleware](https://github.com/modelcontextprotocol/go-sdk/blob/3b917b466cc540079b82bcd0fecdc46a2ba13644/mcp/client.go) already wraps protocol calls, exposing results and errors and allowing early returns. List order and repeated wrapping are documented; inspection of the final mixed chain is not established. Add duplicate checks and verify custom notifications, notification mutation, and reverse requests.

5. **Rust**

   - [Custom request and notification dispatch](https://github.com/modelcontextprotocol/rust-sdk/blob/fd7811fdaa9fefa1c8034534b4d7a31c97204f89/crates/rmcp/src/handler/server.rs) exists on both sides, but does not provide an extension registry.
   - [Service wrappers](https://github.com/modelcontextprotocol/rust-sdk/blob/fd7811fdaa9fefa1c8034534b4d7a31c97204f89/crates/rmcp/src/service.rs) can wrap received calls, observe errors, and return early; adapters are needed for full directional coverage. The public service trait also permits notification wrappers, but end-to-end notification filtering was not established in this review.

6. **Java**

   - [Low-level session constructors](https://github.com/modelcontextprotocol/java-sdk/blob/183935bf80dcb5c70bc13cd7b1eef99ce7053ec3/mcp-core/src/main/java/io/modelcontextprotocol/spec/McpServerSession.java) accept request and notification handler maps and expose `sendRequest`/`sendNotification`. An application can supply wrappers around those handlers through this public constructor; high-level handler preparation remains private.
   - [Capability records](https://github.com/modelcontextprotocol/java-sdk/blob/183935bf80dcb5c70bc13cd7b1eef99ce7053ec3/mcp-core/src/main/java/io/modelcontextprotocol/spec/McpSchema.java) lack `extensions`; message records expose `_meta`. Extension-capability support needs a change; extension-ID registration and high-level middleware integration need further review.

7. **Ruby**

   - [Custom method registration](https://github.com/modelcontextprotocol/ruby-sdk/blob/f22ce86e976be6c71b15c386d5b42c12e5f32ee0/lib/mcp/server.rb) rejects existing names; the client exposes generic requests.
   - [Extension capability support](https://github.com/modelcontextprotocol/ruby-sdk/blob/f22ce86e976be6c71b15c386d5b42c12e5f32ee0/docs/_extensions/capability-extensions.md) is present. General mutation middleware and full notification coverage remain unverified; instrumentation callbacks alone are insufficient.

8. **PHP**

   - [Extension registration](https://github.com/modelcontextprotocol/php-sdk/blob/16836d4e9a0f96831789ac6e64d5ec5238d2c833/src/Server/Builder.php) rejects duplicate IDs and some method conflicts, but custom handlers can override built-ins.
   - [Custom handlers](https://github.com/modelcontextprotocol/php-sdk/blob/16836d4e9a0f96831789ac6e64d5ec5238d2c833/docs/advanced/custom-handlers.md) replace handling rather than wrapping it. Complete the conflict checks and verify failure rollback, general middleware, and capability-setting conflicts.

9. **Kotlin**

   - [Custom request/notification handlers](https://github.com/modelcontextprotocol/kotlin-sdk/blob/1d04427cff47b31932409b90959869aa2934d680/kotlin-sdk-core/src/commonMain/kotlin/io/modelcontextprotocol/kotlin/sdk/shared/Protocol.kt) replace previous handlers. Add checked registration and verify general mutation coverage.
   - [Capability models](https://github.com/modelcontextprotocol/kotlin-sdk/blob/1d04427cff47b31932409b90959869aa2934d680/kotlin-sdk-core/src/commonMain/kotlin/io/modelcontextprotocol/kotlin/sdk/types/capabilities.kt) include extension maps; metadata uses `JsonObject`.

10. **Swift**

    - [Generic method handlers](https://github.com/modelcontextprotocol/swift-sdk/blob/a0ae212ebf6eab5f754c3129608bc5557637e605/Sources/MCP/Server/Server.swift) replace by name; notification handlers append. Client equivalents exist.
    - [Client method/notification APIs](https://github.com/modelcontextprotocol/swift-sdk/blob/a0ae212ebf6eab5f754c3129608bc5557637e605/Sources/MCP/Client/Client.swift) do not establish a general mutation chain. Capability models lack `extensions`; add capability support and review middleware coverage separately.

**Remaining verification**

- All SDKs need conformance checks for extension/core dependencies, standalone and mixed middleware registration, execution order inspection, and failure rollback; source-level partial support is recorded above.
- RPC support must be checked in every permitted direction, and metadata preservation through every codec and convenience API.
- Run shared and SDK-local scenarios against released versions. The table records source findings; no SDK is claimed to pass the full proposal yet.
