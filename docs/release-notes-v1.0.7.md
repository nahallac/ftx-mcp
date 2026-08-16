# ftx-mcp v1.0.7 — release notes

Theme: **display names that work — and a crash class closed.** An agent
asked the bridge to set an object's `DisplayName` and the FactoryTalk
Optix Studio process died with an access violation. This release closes
that crash class at both ends of the wire, then makes both intents
actually work: `DisplayName` is settable through a dedicated attribute
route, and renaming gets a first-class `rename` op.

## Fixed — setting `DisplayName` (or any node attribute) crashed Studio

Crash confirmed live 2026-08-16: `set_property` with `name=DisplayName`
terminated the Studio process (`0xC0000005`, all unsaved edits lost).

The bridge's `DeclaredPropertyGuard` exists precisely to stop this class
of crash: `GetOrCreateVariable` happily fabricates an orphan UA variable
for a property the type does not declare, and Studio's renderer then
dereferences the orphan off-thread and access-violates. The guard's
acceptance test, however, was "any public CLR property on the node's
proxy type" — and UA node **attributes** (`DisplayName`, `BrowseName`,
`Description`, `NodeId`, `NodeClass`) are declared as CLR properties on
the `UAManagedCore` base types of *every* node. They are not UA child
variables, so they passed the gate, got materialized as orphans, and
Studio died on the next render. The legend/`describe_*` filter already
excluded them from the advertised valid set — the guard accepted names it
never listed.

**Fix, bridge side (`studio-bridge/StudioMCPBridge.cs`):**

- The guard's CLR-property match now requires the property to be declared
  in an `FTOptix.*` namespace — the same test the legend filter uses, so
  the gate and the valid list finally agree. Node attributes can no
  longer reach variable materialization through **any** author path:
  `set_property`, `bind`, `attach_expression`, and batch validation all
  share this guard.
- Attribute names get a targeted `node_attribute_not_settable` error
  (instead of a generic `unknown_property`) that names the working paths.

**Fix, service side (`service/core.py`):** the same refusal fires
**before dispatch** in the bind and attach-expression paths, and
`set_property` routes `DisplayName` to the new attribute endpoint (below)
instead of the crash-capable property route. The bridge source is
hand-pasted into a Studio NetLogic node, so a stale bridge is a live
possibility — an older bridge can never see a crash-capable request, and
a bridge without the new endpoint answers with a clean per-op
`not_found`, never a crash.

## New — `DisplayName` is now settable

`set_property` with `name=DisplayName` works: the service routes it to a
dedicated bridge route (`/bridge/node/displayname`, bridge **1.0.7**)
that assigns the node's real `DisplayName` attribute as a
`LocalizedText(value, locale)` — a direct attribute write, never the
variable-materialization path that crashed. Works standalone and inside
`optix_bridge_edit` batches (validation knows the special case).

Notes:

- `BrowseName` stays rename-only: it is the node's identity (paths,
  links, and bindings key on it), so it changes only through the
  re-authoring `rename`/`move` machinery.
- `DisplayName` and the tree label are different things — Studio's
  project tree may label nodes by BrowseName. If you want the visible
  tree name changed, use `rename`.
- `bind` / `attach_expression` on `DisplayName` are still refused — an
  attribute cannot carry a DynamicLink or converter.

## New — `rename` op

Renaming no longer requires knowing the move-with-`new_name` trick.
`optix_bridge_edit` now accepts:

```json
{"op": "rename", "path": "UI/Screens/Foo", "new_name": "Bar"}
```

It is sugar: the service lowers it to the `move` op with the node's own
parent as `new_parent` before validation, so the bridge (any version)
sees a verb it already knows. Move re-authors a copy and deletes the
original (the only mutation pattern proven safe against the 2026-07-17
re-parenting crash class), so the renamed node has a **new NodeId**:
outbound links are re-created, inbound references from elsewhere are not
rewritten. Renaming a top-level node (no parent) is refused, as is a
rename to the same name.

## Upgrading

`pip install --upgrade ftx-mcp` updates the service side. The bridge
changes require re-arming: paste the new
`studio-bridge/StudioMCPBridge.cs` into the `StudioMCPBridge` NetLogic
node, then `StopBridge` / `StartBridge`. `optix_bridge_status` should
report bridge version **1.0.7**.
