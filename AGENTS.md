# AGENTS.md

## Purpose

This file gives coding agents the baseline instructions for working in this repository. It is intentionally practical and should be updated as the project evolves.

UCR is a Windows input/output remapping application with plugin-based device support and an IOWrapper-based backend. The current repository contains the **legacy version**. Future product work should move toward a modern **.NET 10** implementation using **WPF**, **Fluent UI for Windows**, **Microsoft.Extensions dependency injection**, and a testable runtime architecture. The legacy code is kept for reference and feature-parity validation only until the .NET 10 version is on par with existing capabilities; after feature parity is reached and accepted, the legacy implementation should be removed deliberately.

IOWrapper was historically maintained as a separate backend/submodule. Going forward, IOWrapper should be absorbed into the UCR repository for maintenance simplicity, while preserving the backend/provider abstraction that keeps UCR independent from any single hardware API or driver.

## Strategic direction

### Modernization lifecycle

The project is moving through a staged modernization:

1. **Reference stage** - keep the existing legacy application available as the behavioral reference.
2. **Build-out stage** - implement the new .NET 10 version in modern projects while comparing behavior against the legacy app and absorbing IOWrapper backend code into the repository.
3. **Parity stage** - track and close feature gaps until the .NET 10 version is functionally on par with the legacy version.
4. **Removal stage** - after maintainers accept feature parity, remove the legacy implementation and its legacy-only build/test infrastructure in a focused cleanup.

During the reference and build-out stages:

- Treat legacy code as a source of truth for existing behavior, profile semantics, plugin behavior, provider integration details, and migration tests.
- Do not use legacy architecture as the preferred design for new .NET 10 code.
- Do not add new product features to legacy unless explicitly requested for a critical maintenance reason.
- Prefer implementing new capabilities in the .NET 10 version and documenting any intentional behavior differences.
- Maintain a parity checklist or tracking issue/doc once the modern solution exists.

Do not remove legacy code, legacy project files, or legacy build infrastructure until feature parity has been explicitly accepted by maintainers. When removal is authorized, do it as a focused change with migration notes, docs updates, and CI cleanup.

### Legacy baseline

The existing solution is legacy code. Treat it as a working historical implementation and reference implementation, not as the preferred architecture for new work.

- Do not rewrite, retarget, rename, reorganize, or remove legacy projects unless the task explicitly asks for it.
- Keep legacy fixes narrow, low-risk, and compatible with the existing solution structure.
- Preserve behavior for existing profiles, plugin loading, input/output mapping, and device-provider integration.
- Prefer reading legacy code to understand behavior, then implementing cleanly in the .NET 10 architecture.
- When touching legacy code, match the existing language level, framework constraints, project style, and build process.
- Prefer adding tests around behavior before changing remapping, profile, plugin, or device-management logic.

### .NET 10 future version

New product direction should favor a separate .NET 10 architecture rather than incremental modernization inside the old projects.

- Use SDK-style projects for new .NET 10 work.
- Prefer `net10.0` for platform-neutral libraries and `net10.0-windows` only for Windows-specific UI, driver, HID, or device-provider code.
- Use WPF for the Windows desktop application and a Fluent UI for Windows control/theme stack for the modern user interface.
- Use `Microsoft.Extensions.Hosting` / `Microsoft.Extensions.DependencyInjection` as the application composition model.
- Keep domain logic independent from WPF, Fluent UI controls, XAML, and hardware drivers.
- Isolate Windows/device-specific integrations behind adapters or provider interfaces.
- Enable nullable reference types, analyzers, and modern async/cancellation patterns in new projects.
- Prefer dependency injection and explicit dependencies over hidden global state.
- Design every service boundary so it can be unit tested with mocks/fakes.
- Design the runtime model so recorded input traces can be replayed and compared against expected output recordings.
- Design the new version so profile/config migration from the legacy version is testable and documented.
- Use the legacy version as the feature reference until the .NET 10 version reaches accepted feature parity.
- Track feature parity explicitly across profiles, plugin loading, remappers, filters, providers, UI workflows, import/export, and runtime behavior.
- Do not introduce broad compatibility breaks to plugins, profiles, or provider concepts without documenting the migration path.
- Do not delete legacy code as part of modernization work until parity has been accepted and removal is the explicit task.

If a new project layout is introduced, prefer a clear modern structure such as:

```text
src/
  UCR.App.Wpf/              # WPF + Fluent UI shell, composition root, views
  UCR.App.Abstractions/     # UI-facing app contracts that are not tied to WPF controls
  UCR.Core/                 # profiles, mappings, filters, variants, controller groups
  UCR.Runtime/              # activation planner, runtime sessions, replayable input/output pipeline
  UCR.Plugins.Abstractions/ # plugin contracts and metadata
  UCR.Plugins.BuiltIn/      # built-in remappers and filters
  UCR.IO.Abstractions/      # provider/device/binding/subscription contracts
  UCR.IO/                   # backend controller, provider registry, subscription routing
  UCR.IO.Providers.Windows/ # DirectInput, XInput, virtual devices, MIDI, Tobii, Interception, etc.
  UCR.Migration/            # legacy context/profile import and compatibility helpers
  UCR.Cli/                  # optional command-line activation/control front-end
tests/
  UCR.Core.Tests/
  UCR.Runtime.Tests/
  UCR.IO.Tests/
  UCR.Migration.Tests/
  UCR.App.Wpf.Tests/
  UCR.Runtime.ReplayTests/
legacy/
  # optional temporary home for the existing .NET Framework solution, if it is moved intentionally before final removal
```

Do not create this structure unless the task calls for it or an approved modernization plan exists. If a `legacy/` folder is introduced, treat it as a temporary reference location, not a permanent supported product line.


### Future product vision and goals

The .NET 10 version should become a redesigned UCR, not a direct mechanical port. Preserve legacy behavior where needed for user migration and parity checks, but build the future model around explicit activation composition, controller groups, variants, shareable mappings, and runtime-safe provider abstractions.

Primary goals:

- Make the current .NET Framework code and legacy IOWrapper behavior the reference, not the future architecture.
- Build the modern app as a .NET 10 WPF application using Fluent UI for Windows.
- Use .NET dependency injection for all application, domain, runtime, provider, plugin, migration, and UI services.
- Make the runtime model replayable and unit-testable with mocked services, fake providers, real-time input traces, and expected output recordings.
- Absorb IOWrapper into the UCR repository so backend/provider maintenance happens with the app and plugin model.
- Replace implicit editable profile nesting with explicit profile activation composition.
- Make controller selection, controller replacement, and layout remapping first-class concepts.
- Introduce variants as the semantic input/output abstraction above DirectInput, XInput, and other raw providers.
- Keep profiles more shareable by mapping to named variant controls instead of raw provider-specific indexes whenever possible.
- Preserve per-binding preview and add richer variant-level visual previews for both input and output devices.
- Make runtime activation deterministic, inspectable, testable, and safe to roll back when provider subscriptions fail.

#### Future profile and activation model

The future profile model intentionally changes the legacy nested-profile concept.

- Profiles should be independent activation units. The primary model should not rely on editable nested child profiles as the way to inherit behavior.
- A profile may declare **Additional profile activations**: other profiles that should be activated alongside it.
- The final product name for this feature is not fixed. Until maintainers choose a name, use **Additional profile activations** in code comments and docs.
- Additional activations should not recursively chain by default. If profile `A` activates profile `B`, then profile `B`'s own additional activations should load only when an explicit setting allows chaining such as `Load additional activations from activated profiles`.
- Activation composition must detect cycles, duplicate activations, and ambiguous ordering.
- The UI may show additional/nested profiles as read-only in the context of the parent activation. Editing should happen in the source profile's own editor unless maintainers approve an inline-edit model.
- Plugin activation for additional/nested profiles must be controlled deliberately. Support an advanced setting that allows a plugin or activation to opt out of running when its owning profile is loaded as an additional/nested activation.
- Keep activation state observable: users and diagnostics should be able to see which profiles were requested, which additional profiles were loaded, which were skipped, and why.

Migration guidance:

- Legacy child-profile inheritance should migrate to explicit activation composition only when the resulting behavior is clear and test-covered.
- If a legacy nested profile cannot be represented safely, keep the original data, warn during migration, and document the manual fix.
- Do not silently flatten profile trees in a way that changes mapping override, filter, shadow-device, or plugin activation behavior.

#### Future profile controller groups

Controller groups should become the main way a profile describes the controllers it needs.

- A **profile controller group** is a named set of controller slots selected by a profile.
- A profile should be able to override its controller selections without modifying other profiles that use the same devices or variants.
- A **shadow group** inherits from another controller group but can override selected controllers. Use this for duplicate controllers, secondary devices, mirrored layouts, or user-specific substitutions.
- Groups should be eligible for additional/nested profile activation only when allowed. Provide a way to disable a group from being loaded through additional profile activation.
- Group activation should be explicit in the runtime activation plan so agents can test and diagnose which controllers each profile actually uses.
- Do not treat controller groups as UI-only constructs; they are part of the domain model for runtime binding and migration.

#### Future variant-based input and output management

DirectInput, XInput, and similar provider APIs should not be the user-facing mapping surface in the .NET 10 version. Providers should expose raw capabilities; **variants** should provide the semantic abstraction that profiles bind to.

A variant is a named interpretation of a provider/device's controls:

- It aliases raw provider controls into user-facing semantic controls.
- The default variant should mimic the provider's normal naming so existing expectations still work.
- XInput variants might expose controls such as `Button A`, `Button B`, `Left Trigger`, and `Right Stick X`.
- Accessibility variants may rename controls by how a device is physically used, such as `Tongue`, `Sip`, `Puff`, `Left Paddle`, or other user-specific labels.
- DirectInput variants may represent different arcade layouts such as 6-button, 8-button, racing wheel, flight stick, or custom cabinet layouts where button order matters.
- Variants should apply to both input and output devices where useful.
- A profile should choose a concrete device and a mapping variant rather than binding directly to raw DirectInput/XInput indexes.

Variant goals:

- Make profiles more portable and shareable across devices with equivalent layouts.
- Enable device replacement by remapping a device to the same variant contract.
- Enable global remapping by changing the variant/controller map rather than editing every profile mapping.
- Support user-defined variants and visuals through a versioned config format. XML is acceptable if chosen deliberately; any schema must be documented, validated, and migration-tested.
- Allow each variant to define visual representations for inputs and outputs, including animated previews where practical.
- Preserve per-binding previews even when richer whole-device previews exist.

Do not hard-code XInput or DirectInput layout assumptions into plugins. Plugins should operate on UCR's normalized control categories and values, with variant/provider adapters translating raw device state into those controls.

#### Future mapping and controller-map model

Mappings should target enumerated controller controls exposed by variants and controller groups.

- Controllers should expose stable enumerated controls such as `Button 1`, `Button 2`, `Axis 1`, `Axis 2`, and provider/variant-specific display names.
- The provider or selected variant may supply names, labels, grouping, and visual positions for those controls.
- A controller map should be able to remap controls, for example swapping `Button 1` and `Button 3`, without rewriting every profile mapping.
- Mapping through variants should support device replacement and global remapping while keeping profile mappings stable.
- Runtime remapping control, such as a named-pipe control channel, may be useful later, but it must be isolated behind a service/transport boundary. Do not make named pipes or any specific IPC transport part of the core mapping model unless maintainers explicitly choose that design.

#### Future profile UI goals

The profile editor should make runtime composition understandable.

- A profile box or equivalent UI should show the active profile, additional profile activations, controller groups, selected variants, and any read-only nested/additional profiles.
- Read-only additional profiles should be visually distinct from editable profile content.
- Users should be able to inspect why a profile, plugin, group, or controller was active, skipped, inherited, shadowed, or overridden.
- Variant visuals should support input and output previews. Per-binding previews must remain available for precise troubleshooting.
- Do not finalize a profile-box implementation from incomplete notes alone; keep the model flexible until maintainers approve the UX.

#### Future CLI and runtime goals

CLI activation should be preserved and expanded for automation.

- Support activating a profile by name/path.
- Support a fallback profile when the requested profile is missing, for example conceptually: `Activate Tetris --fallback GameBoy`.
- CLI activation should return a clear success/failure code and explain whether the primary profile, fallback profile, or neither profile was activated.
- CLI/runtime activation should share the same activation planner as the UI so behavior does not diverge.
- Activation diagnostics should include requested profile, fallback profile, additional activations, chained activation decisions, controller groups, variants, provider subscriptions, and skipped plugins/groups.

### Future implementation specification

This section is the baseline specification for the .NET 10 implementation. It resolves the current vision notes and open questions into concrete default decisions for agents. Maintainers can still change these decisions later, but agents should not leave implementation choices ambiguous when working from this file.

#### Specification status

- **Target product:** modern UCR for Windows, built on .NET 10, WPF, Fluent UI, dependency injection, and a replayable runtime model.
- **Legacy role:** reference implementation and migration source only.
- **Backend direction:** IOWrapper concepts are absorbed into this repository as in-repo provider/runtime projects.
- **Testing contract:** services are unit tested through DI/mocking; runtime behavior is tested with deterministic replay fixtures and expected output recordings.
- **Feature-parity gate:** legacy code is removed only after the .NET 10 implementation reaches accepted feature parity.

#### Resolved decisions, one by one

| # | Area / open question | Decision | Implementation contract |
| --- | --- | --- | --- |
| 1 | Platform | Use .NET 10. | New projects use SDK-style projects. Libraries that are not Windows-specific target `net10.0`; WPF, Windows provider, HID, driver, and virtual-device projects target `net10.0-windows`. |
| 2 | Desktop UI | Use WPF. | `UCR.App.Wpf` is the Windows desktop shell. No domain, runtime, plugin, provider, migration, or test project should depend on WPF unless it is explicitly UI-only. |
| 3 | Fluent UI | Use Fluent UI for Windows styling in WPF. | Baseline package choice is `WPF-UI` for Fluent-style WPF controls unless maintainers replace it. Wrap navigation, dialogs, theme, accent color, and notifications behind UI services so the library can be swapped. |
| 4 | App composition | Use the .NET Generic Host and built-in dependency injection. | The WPF app is the composition root. Remove `StartupUri`, create the host during app startup, register services in extension methods, resolve the main window from DI, and stop/dispose the host on app exit. |
| 5 | Build system | Use GitHub Actions and standard `dotnet` CLI commands. | Future CI calls `dotnet restore`, `dotnet build`, `dotnet test`, and `dotnet publish`. NUKE remains legacy-maintenance only. |
| 6 | Legacy code | Keep existing code as reference-only. | Do not add normal product features to legacy. Use it to compare behavior, import configs, and write parity tests. Delete it only after explicit parity acceptance. |
| 7 | IOWrapper | Absorb IOWrapper into UCR. | Move the backend/provider architecture into in-repo .NET 10 projects. Preserve provider abstraction, subscription routing, bind mode, input/output separation, and capability reporting. |
| 8 | Plugin discovery | Replace legacy MEF-first design with a DI-friendly catalog. | Built-in plugins are normal in-repo types registered through `IPluginCatalog`/`IPluginFactory`. External plugin loading is deferred until the modern contract is stable. |
| 9 | Profiles | Profiles are independent activation units. | The future data model must not depend on editable child profiles as the primary inheritance mechanism. |
| 10 | Nested profiles | Do not implement legacy-style editable nesting as the primary model. | Legacy nested profiles may appear read-only during migration/inspection. New composition uses additional profile activation links. |
| 11 | Feature naming | Use **Additional profiles** in UI and `AdditionalProfileActivations` / `ProfileActivationLink` in code. | Avoid calling the core model “nested profiles”. Use “nested” only for legacy import, read-only display, or explanatory docs. |
| 12 | Additional activation behavior | A profile may activate other profiles with it. | The root requested profile plus its additional profiles form one activation plan and one runtime session. |
| 13 | Activation chaining | Chaining is off by default. | If profile `A` activates profile `B`, profile `B`'s own additional profiles are ignored unless `LoadAdditionalProfilesFromAdditionalProfiles` is explicitly enabled for that link or activation request. |
| 14 | Activation cycles | Cycles are invalid. | Detect cycles before touching providers. Return diagnostics that name the cycle path. |
| 15 | Duplicate profiles | Duplicate activation links collapse to one profile instance. | Keep deterministic ordering and record a diagnostic such as “Profile X already included; duplicate skipped.” |
| 16 | Activation ordering | Compose additional profiles first, root profile last. | Additional profiles are applied in the order listed. The root profile has highest precedence so the explicitly activated profile can override shared/base behavior. |
| 17 | Chained activation ordering | Chained profiles load immediately before the profile that requested them. | This keeps common/shared profiles lower precedence than the profile that references them. |
| 18 | Profile editing | Additional profiles are read-only when shown inside another profile's editor/runtime view. | Provide navigation/open-source-profile actions instead of inline editing by default. |
| 19 | Plugin activation when additional | Plugins run when their owning profile is included, unless explicitly disabled for additional activation. | Add an advanced setting such as `DisableWhenLoadedAsAdditionalProfile` on plugin instance or activation metadata. Default is **false** so existing profile behavior is preserved. |
| 20 | Group activation when additional | Controller groups can opt out of additional-profile loading. | Add `IncludeWhenLoadedAsAdditionalProfile` on controller groups; default is **true**. If false, skip the group and all mappings that require it when the profile is additional. |
| 21 | Controller groups | Controller groups are first-class domain objects. | A profile declares named groups such as `Player1Gamepad`, `ArcadePanel`, `Wheel`, or `AccessibilityController`. Mappings target controller groups, not raw devices. |
| 22 | Controller slots | Groups contain stable controller slots. | A slot represents the logical device role used by mappings. Slots are bound to provider devices plus variants at activation time. |
| 23 | Controller overrides | Profiles can override controller assignments. | The root profile may override controllers declared by additional profiles. Overrides must be explicit, visible in diagnostics, and covered by tests. |
| 24 | Shadow groups | Shadow groups inherit from another controller group and can override selected controllers. | Use shadow groups for duplicate controllers, secondary devices, split-player setups, and device substitutions. Shadow group identity must be visible in activation diagnostics. |
| 25 | Shadow group additional loading | Shadow groups can be disabled when loaded from additional profiles. | Add a setting such as `IncludeShadowGroupWhenLoadedAsAdditionalProfile`; default is **true** unless the group is marked unsafe for composition. |
| 26 | Raw provider mapping | Profiles should not bind directly to DirectInput/XInput raw indexes. | Providers expose raw controls, but profile mappings bind to variant controls. Raw ids are stored only in controller maps/device assignments. |
| 27 | Variants | Variants are the semantic abstraction over provider devices. | A variant maps provider controls to stable semantic controls, names, groups, display labels, and visuals. |
| 28 | Default variants | Every supported device/provider family must have a default variant. | Defaults mimic ordinary provider naming: XInput `A/B/X/Y`, DirectInput `Button 1`, `Axis X`, etc. |
| 29 | User variants | Support user-defined variants. | User variant packs use a constrained, versioned XML format by default. They must validate against a schema and must not contain executable code or arbitrary XAML. |
| 30 | Accessibility variants | Variants can rename controls around device use rather than hardware labels. | Example: an XInput-compatible accessibility device may expose `Tongue`, `Sip`, or `Puff` controls mapped to raw XInput buttons. |
| 31 | Arcade variants | Variants can represent different physical layouts for the same provider type. | Example: DirectInput arcade panel variants for 6-button, 8-button, Sega-style, noir, or custom layouts. |
| 32 | Variant visuals | Variants may define safe visual layouts for input and output previews. | Use declarative metadata rendered by trusted WPF controls. Do not load arbitrary XAML from variant packs. |
| 33 | Animated previews | Variant visuals may be animated. | Animation is driven by runtime/control state and safe visual primitives. Keep animation optional and test visual-state mapping separately from WPF rendering. |
| 34 | Per-binding preview | Preserve per-binding preview. | Whole-device variant previews supplement, not replace, precise per-binding state inspection. |
| 35 | Controller maps | Controller maps remap semantic controls without rewriting profiles. | Example: swap `Button 1` and `Button 3` for a controller slot. Controller maps can support device replacement and global remapping. |
| 36 | Mapping target | Mappings target stable variant controls and controller slots. | Display names are not stable ids. Persist ids; localize/rename labels separately. |
| 37 | Runtime IPC | Named pipes are optional infrastructure, not core architecture. | If runtime control is needed, define `IRuntimeControlEndpoint`; named pipes can be one implementation in an infrastructure project. |
| 38 | CLI | Add a DI-hosted CLI for activation and diagnostics. | CLI commands call the same planner/services as the UI. No duplicated activation logic. |
| 39 | CLI fallback | Fallback profile is used when the requested profile is missing. | `ucr activate Tetris --fallback GameBoy` activates `GameBoy` only if `Tetris` cannot be resolved. Activation failure fallback is a separate future option and must not be implied. |
| 40 | CLI result codes | CLI returns deterministic exit codes. | `0` success, `1` validation/user error, `2` activation/runtime failure, `3` provider/device unavailable, `4` unexpected/internal error. |
| 41 | Profile box UI | Use a Fluent UI composition card/panel as the baseline. | It shows requested profile, active profile, fallback status, additional profiles, read-only profile layers, controller groups, variants, overrides, skipped groups/plugins, and provider health. |
| 42 | Persistence format | Use versioned JSON for modern app profiles/settings; XML for user variant packs unless changed by maintainers. | Keep legacy `context.xml` import. Use explicit schema versions, validation, migration reports, and tests. |
| 43 | Modern serialization | Prefer `System.Text.Json` for modern JSON. | Use source generation where helpful. Do not carry Newtonsoft.Json forward unless a compatibility case requires it. |
| 44 | Profile identity | Profiles have stable ids separate from names/paths. | Names can change; ids are used for links and references. CLI may resolve by path/name but stores stable ids internally. |
| 45 | Runtime value model | Use strongly typed normalized control values. | Axes use `double` in `[-1.0, 1.0]`; buttons use `bool` or `0/1`; deltas use signed numeric deltas; events use pulse/event records; POV/hat uses neutral plus angle/direction. Preserve raw metadata separately. |
| 46 | Legacy short range | Keep legacy `short` axis semantics at adapters/migration boundaries. | Legacy algorithms using `-32768..32767` should be ported with tests and then adapted to normalized values deliberately. |
| 47 | Runtime execution | Use a deterministic ordered runtime event loop per active session. | Provider callbacks enqueue events; the runtime processes them in timestamp/order sequence. No WPF or provider blocking on the hot path. |
| 48 | Activation transaction | Activation is transactional. | Validate first; subscribe outputs before inputs; commit active session only after subscriptions succeed; roll back in reverse order on failure. |
| 49 | Recording | Runtime can record input and output streams. | Recordings include activation plan id, devices, variants, input events, output events, timestamps, and diagnostics. |
| 50 | Replay tests | Runtime behavior is validated with replay fixtures. | Tests feed timestamped inputs through fake providers and compare expected output recordings. No sleeps, real hardware, drivers, or admin rights in normal CI. |
| 51 | Test framework | Use NUnit for modern tests unless maintainers explicitly choose otherwise. | This preserves continuity with legacy tests while still using `dotnet test`. Do not mix test frameworks inside the same modern test project. |
| 52 | Mocking | Use NSubstitute as the default mocking library. | Prefer fakes for high-volume runtime streams and mocks for service interactions/failure paths. |
| 53 | Assertions | Use clear domain assertions; FluentAssertions is allowed if adopted consistently. | Assertion style must prioritize readable replay failures and diagnostics. |
| 54 | UI testing | Unit test view models and UI services, not WPF rendering in normal CI. | Optional visual/snapshot/manual UI tests must be separated from default CI. |
| 55 | Provider tests | Provider adapters must have contract tests with fake devices where possible. | Hardware-specific/manual tests must be opt-in and tagged. |
| 56 | Parity | Feature parity is tracked explicitly. | Parity must cover migration, activation, variants, mappings, filters, plugin behavior, shadow groups, provider routing, bind mode, CLI, UI workflows, and replay recordings. |

#### Product domain model

The future model should be explicit and composable. Use these concepts consistently.

| Concept | Description | Must be persisted? | Runtime role |
| --- | --- | --- | --- |
| `Profile` | Independent user configuration that can be activated. | Yes | Root or additional activation unit. |
| `ProfileActivationLink` | Link from one profile to another profile to activate with it. | Yes | Builds profile composition. |
| `ActivationRequest` | Request from UI, CLI, IPC, startup, or tests. | No, except logs/history | Input to planner; may include fallback and options. |
| `ActivationPlan` | Fully resolved, deterministic plan before touching providers. | Diagnostic/replay artifact | Single source of truth for runtime session creation. |
| `RuntimeSession` | Active execution instance. | No | Owns subscriptions, event routing, filter states, output sinks. |
| `ControllerGroup` | Named collection of controller slots used by a profile. | Yes | Scopes mappings and device assignment. |
| `ControllerSlot` | Logical controller role such as `Player1Gamepad`. | Yes | Binds mappings to real devices via assignments. |
| `ControllerAssignment` | Selected provider device and variant for a slot. | Yes | Resolves semantic controls to provider controls. |
| `ShadowGroup` | Group derived from another group with optional overrides. | Yes | Enables duplicate/secondary controller behavior. |
| `Variant` | Semantic layout over a provider/device family. | Yes, from built-in or user packs | Maps raw controls to stable semantic controls and visuals. |
| `ControllerMap` | Remapping layer between semantic controls. | Yes | Enables swapping/replacing controls globally or per profile. |
| `Mapping` | User-defined transformation from inputs to outputs. | Yes | Runtime route through plugin/filter pipeline. |
| `PluginInstance` | Configured remapper/filter/action. | Yes | Executes transformation logic. |
| `FilterState` | Runtime state used to gate plugin execution. | Optional if persisted by profile | Determines whether plugins/mappings run. |
| `InputEvent` | Timestamped normalized input change. | In recordings | Runtime input to pipeline. |
| `OutputEvent` | Timestamped normalized output command. | In recordings | Runtime output from pipeline. |
| `ProviderDevice` | Concrete device discovered by a provider. | Assignments/cache | Hardware/API endpoint. |
| `BindingDescriptor` | Provider/raw control identity. | Assignments/maps | Adapter boundary only. |

#### Dependency injection and service boundaries

The .NET 10 platform must use dependency injection for application services. Constructor injection is the default. Static mutable globals are prohibited for runtime state.

Required DI rules:

- Register all services through project-local extension methods such as `services.AddUcrCore()`, `services.AddUcrRuntime()`, `services.AddUcrIo()`, `services.AddUcrPlugins()`, `services.AddUcrWpfUi()`, and `services.AddUcrMigration()`.
- Use `ILogger<T>`, `IOptions<T>`, configuration, and `TimeProvider` instead of direct global logging/config/time access.
- Do not pass `IServiceProvider` into domain/runtime services. Service location is allowed only in composition roots, factories, plugin factories, and WPF navigation/dialog factories.
- Do not create provider, runtime, profile, serializer, file-system, clock, dispatcher, dialog, navigation, or settings services with `new` inside business logic.
- Register factories for objects that require runtime parameters, for example `IRuntimeSessionFactory`, `IPluginInstanceFactory`, or `IActivationScopeFactory`.
- All services that own native handles, provider subscriptions, channels, file watchers, timers, or background tasks must implement `IDisposable` or `IAsyncDisposable` as appropriate.
- Use `CancellationToken` for long-running operations, provider discovery, activation, replay, migration, file IO, and hosted services.

Required service groups:

| Service area | Required interfaces | Lifetime guidance | Testing requirement |
| --- | --- | --- | --- |
| Profiles | `IProfileRepository`, `IProfileSerializer`, `IProfileValidator`, `IProfileMigrationService`, `IProfileImportService` | Singleton serializers; scoped editing/import sessions | Mock file system; test versioning, validation, import warnings. |
| Activation | `IProfileActivationPlanner`, `IActivationPlanValidator`, `IProfileActivationService`, `IActivationDiagnosticsWriter` | Scoped per activation request | Unit test every planner branch and rollback path. |
| Runtime | `IRuntimeEngine`, `IRuntimeSession`, `IRuntimeSessionFactory`, `IInputEventRouter`, `IOutputEventSink`, `IRuntimeRecorder`, `IRuntimeReplayService` | Session-scoped runtime; singleton factories | Replay tests for deterministic event-to-output behavior. |
| Controllers | `IControllerGroupService`, `IControllerAssignmentService`, `IShadowGroupResolver`, `IControllerMapService` | Singleton/stateless or scoped per plan | Test overrides, shadow inheritance, disabled additional loading. |
| Variants | `IVariantRegistry`, `IVariantResolver`, `IVariantPackLoader`, `IVariantVisualService`, `IVariantValidator` | Registry singleton; loaders scoped/transient | Test default/user variants, schema validation, visual metadata. |
| Providers | `IProviderRegistry`, `IDeviceDiscoveryService`, `IInputProvider`, `IOutputProvider`, `IBindModeProvider`, `IProviderHealthService` | Provider registry singleton; provider instances owned/disposable | Contract tests with fake devices and error injection. |
| Plugins | `IPluginCatalog`, `IPluginFactory`, `IPluginMetadataService`, `IPluginRuntimeAdapter`, `IPluginValidator` | Catalog singleton; instances transient/session-scoped | Test metadata, validation, lifecycle, filters, output writes. |
| CLI/control | `ICommandLineActivationService`, `IRuntimeControlEndpoint`, `IRuntimeStatusService` | Hosted/endpoint services singleton; requests scoped | Test parsing and activation via mocked planner/service. |
| UI | `INavigationService`, `IDialogService`, `INotificationService`, `IThemeService`, `IUiDispatcher`, `IViewModelFactory` | UI shell services singleton; view models transient | Mock UI services; test view models without WPF rendering. |
| Infrastructure | `IFileSystem`, `ISettingsStore`, `IAppVersionProvider`, `IProcessExitService` | Singleton | Replace with fakes in tests. |

#### WPF and Fluent UI specification

The WPF app is a thin presentation layer over DI services.

Required UI architecture:

- Use MVVM for screens, dialogs, profile editor, activation diagnostics, device manager, variant editor, and settings.
- Use Fluent UI controls for shell navigation, profile boxes, cards, settings, dialogs, snackbars/toasts, theme/accent integration, and iconography.
- Use a DI-resolved `MainWindow`, shell view model, page view models, dialogs, and services.
- Keep code-behind limited to WPF event bridging, focus management, drag/drop bridging, and control-specific glue. No activation, mapping, provider, migration, or plugin logic in code-behind.
- Use `IUiDispatcher` to marshal to the UI thread. View models and services must be testable without a WPF dispatcher.
- Do not load user-provided XAML. Render variant visuals from safe primitives such as rectangles, circles, paths from a constrained allowlist, labels, images from trusted locations, and state bindings.
- Theme and accent selection must go through `IThemeService`; do not scatter direct Fluent UI theme calls throughout view models.
- The app shell must expose an activation diagnostics surface because runtime composition can be complex.

Baseline navigation areas:

1. **Dashboard / Profile Box** - current active profile, fallback/additional activation status, active controller groups, variants, provider health, skipped items.
2. **Profiles** - profile list, editor, additional profile links, mappings, plugin instances, filters.
3. **Controllers** - controller groups, slots, assignments, overrides, shadow groups, controller maps.
4. **Variants** - built-in variants, user variants, visual previews, validation.
5. **Devices** - discovered devices, provider health, cached/offline devices, bind-mode tools.
6. **Runtime** - activation history, recordings, replay tools, diagnostics.
7. **Settings** - app, UI theme/accent, provider settings, advanced activation behavior.

#### Activation planning specification

Every activation entry point must call the same planner and execution service.

Activation entry points:

- UI activation.
- CLI activation.
- Startup auto-activation, if implemented.
- Runtime control endpoint activation, if implemented.
- Test/replay activation.

Planner algorithm:

1. Create an `ActivationRequest` containing requested profile reference, optional fallback reference, source, and options.
2. Resolve requested profile by stable id, profile path, or unique name.
3. If the requested profile is missing and fallback is specified, resolve fallback. Record fallback diagnostics.
4. If both requested and fallback profiles are missing, fail before provider access.
5. Build profile composition layers: additional profiles in declared order, chained profiles only when explicitly enabled, root profile last.
6. Detect cycles, duplicate links, missing additional profiles, disabled additional profile links, and ambiguous names.
7. Resolve controller groups for each profile layer.
8. Apply `IncludeWhenLoadedAsAdditionalProfile` rules to groups and shadow groups.
9. Resolve root profile controller overrides over additional profiles.
10. Resolve shadow groups and inherited controller assignments.
11. Resolve device assignments, provider devices, selected variants, and variant control ids.
12. Apply controller maps.
13. Resolve mappings and plugin instances in layer order.
14. Apply plugin participation rules, including `DisableWhenLoadedAsAdditionalProfile`.
15. Validate required provider capabilities: input subscription, output subscription, bind mode, blocking, virtual outputs, haptics, LEDs, or other optional capabilities.
16. Produce a serializable `ActivationPlan` with diagnostics.
17. Execute transactionally: prepare outputs, subscribe outputs, create runtime session, subscribe inputs, then commit active session.
18. Roll back partial subscriptions in reverse order on any failure.
19. Publish activation diagnostics and update UI/CLI/runtime status.

Precedence rules:

- Root profile has highest precedence.
- Later additional profiles override earlier additional profiles only where a model explicitly allows overrides.
- Controller overrides beat inherited/default assignments.
- Shadow group overrides beat inherited group assignments for the shadow group only.
- Controller maps apply after variant resolution and before plugin input routing.
- Filters gate plugin execution after event routing but before plugin update.

#### Profile composition specification

Profiles are independent activation units.

Required behavior:

- A profile can be activated alone.
- A profile can list additional profiles to activate with it.
- Additional profiles are shown read-only in the root profile context.
- Additional profiles can be opened directly for editing.
- Additional activation links can be enabled/disabled.
- Additional activation links can allow/disallow chaining.
- Additional activation links can optionally carry ordering metadata; absent metadata uses list order.
- Additional profiles can contribute controller groups, mappings, filters, and plugins unless those items opt out of additional loading.
- The activation diagnostics must show why each profile layer was included, skipped, duplicated, disabled, or failed.

Migration behavior from legacy nested profiles:

- Legacy parent/child profile trees import as independent profiles plus activation links where that best preserves behavior.
- The importer must produce warnings where legacy override semantics cannot be represented exactly.
- Imported nested profiles should be displayed read-only in composed contexts until opened as their own source profile.
- Migration tests must include multi-level nesting, mapping overrides, inherited devices, filters, and shadow devices.

#### Controller group and shadow group specification

Controller groups decouple profiles from physical devices.

Required objects:

- `ControllerGroupId` - stable id for a group.
- `ControllerGroupName` - display name.
- `ControllerSlotId` - stable id for a logical slot.
- `ControllerAssignment` - selected provider device and variant for a slot.
- `ControllerOverride` - root/profile-specific replacement of an assignment.
- `ShadowGroup` - group derived from another group, with optional overridden assignments and maps.

Required behavior:

- Profiles reference controller groups, not direct device objects.
- Groups can be shared across profiles.
- Root profile overrides can replace assignments from additional profiles.
- Shadow groups inherit slots, variants, maps, and assignments from a base group unless overridden.
- Shadow groups can be included or excluded from additional-profile loading.
- Diagnostics must explain inherited vs overridden controllers.
- Tests must cover missing controllers, duplicate slots, invalid shadow base, override conflicts, and disabled additional loading.

#### Variant and input-management specification

DirectInput, XInput, and other providers are raw input sources. Variants define the user-facing controls.

Required identity layers:

| Layer | Example | Purpose |
| --- | --- | --- |
| Provider id | `xinput`, `directinput`, `midi`, `vjoy` | Identifies backend adapter. |
| Device id | `xinput:0`, HID instance path, provider handle | Identifies discovered device. |
| Device family | `xinput-gamepad`, `generic-di-joystick`, `arcade-panel` | Chooses default variants and capabilities. |
| Provider control id | raw button/axis/POV index | Provider adapter boundary. |
| Variant control id | `ButtonA`, `Button1`, `Tongue`, `AxisX` | Stable semantic mapping target. |
| Display label | `A`, `South`, `Tongue`, `Fire 1` | User-facing text, localizable/editable. |
| Visual element id | `face-button-south`, `axis-left-x` | Preview/animation binding. |

Required behavior:

- Built-in default variants exist for each supported provider/device family.
- User variant packs can define aliases, display labels, grouping, visual layout, and animation metadata.
- Variants can be used for both input and output devices.
- Variants are validated before use; invalid variants do not partially load.
- Profiles persist stable ids, never display names only.
- Variant visuals are optional but recommended for shareability and accessibility.
- Variant mapping makes profiles shareable across physical devices if those devices support the same semantic controls.

#### Mapping, plugin, and filter specification

Mappings describe how normalized inputs become outputs.

Required behavior:

- A mapping belongs to a profile layer and references controller groups/slots and variant controls.
- A mapping can contain one or more plugin instances.
- Plugins declare required input categories and output categories.
- Plugin input categories use normalized categories, not provider-specific types.
- Plugins can write output events and filter-state events.
- Filters are named runtime states that can gate plugin execution.
- Filter names are stable ids and should be case-insensitive for matching.
- Additional-profile and shadow-group behavior must be testable per mapping/plugin.

Built-in plugin parity set:

- Axis to Axis.
- Axes to Axes.
- Axis to Button.
- Button to Axis.
- Buttons to Axis.
- Axis Splitter.
- Axis Merger.
- Axis Initializer.
- Button to Button.
- Button to Event.
- Axis to Filter.
- Button to Filter.

Porting rules:

- Port behavior with tests before refactoring algorithms.
- Preserve legacy edge cases intentionally or document differences.
- Keep plugin logic free of WPF and provider APIs.
- Plugin instances are created through `IPluginFactory` so they can receive services or test doubles when needed.

#### Runtime model and replay testing specification

The runtime must support both real-time device input and deterministic replay.

Runtime event requirements:

- `InputEvent` includes timestamp, sequence number, provider id, device id, controller slot id, variant id, variant control id, normalized value, and optional raw metadata.
- `OutputEvent` includes timestamp, sequence number, output provider/device id, controller slot id, variant control id, normalized value, and optional raw metadata.
- The runtime assigns monotonic sequence numbers to preserve deterministic ordering when timestamps match.
- The runtime event loop processes events through a bounded channel/queue owned by the runtime session.
- Provider callbacks must not execute plugin logic directly. They enqueue input events.
- Output sinks must be mockable and recordable.

Recording requirements:

- Record activation request and activation plan id.
- Record provider/device/variant setup.
- Record input stream.
- Record output stream.
- Record diagnostics and errors.
- Support redaction of user-specific device paths if recordings are shared.

Replay fixture format:

```json
{
  "schemaVersion": 1,
  "name": "xinput-a-to-vjoy-button-1",
  "activation": {
    "profile": "Tetris",
    "fallbackProfile": "GameBoy"
  },
  "devices": [
    {
      "provider": "xinput",
      "device": "xinput-0",
      "family": "xinput-gamepad",
      "variant": "default-xinput"
    },
    {
      "provider": "virtual-gamepad",
      "device": "virtual-gamepad-0",
      "family": "virtual-gamepad",
      "variant": "default-virtual-gamepad"
    }
  ],
  "inputs": [
    { "atMs": 0, "device": "xinput-0", "control": "ButtonA", "value": true },
    { "atMs": 60, "device": "xinput-0", "control": "ButtonA", "value": false }
  ],
  "expectedOutputs": [
    { "atMs": 0, "device": "virtual-gamepad-0", "control": "Button1", "value": true },
    { "atMs": 60, "device": "virtual-gamepad-0", "control": "Button1", "value": false }
  ]
}
```

Replay rules:

- Tests run with fake providers, fake output sinks, and `TimeProvider`/virtual time.
- No sleeps or wall-clock timing in normal tests.
- Exact output ordering is required unless the fixture declares a tolerance.
- Fixtures may assert expected no-output windows.
- Replay tests are required for feature parity claims involving mappings, filters, variants, controller maps, shadow groups, bind mode, fallback activation, and additional profiles.

#### Unit testing and mocking specification

Testing is a first-class design requirement.

Required test projects:

```text
tests/
  UCR.Core.Tests/
  UCR.Runtime.Tests/
  UCR.Runtime.ReplayTests/
  UCR.IO.Tests/
  UCR.Plugins.Tests/
  UCR.Migration.Tests/
  UCR.App.Wpf.Tests/
  UCR.Cli.Tests/
```

Required test categories:

- **Unit tests:** all services and domain models.
- **Contract tests:** provider adapters, plugin contracts, variant pack schema, profile schema.
- **Replay tests:** runtime input/output recordings.
- **Migration tests:** legacy `context.xml` and IOWrapper compatibility cases.
- **UI view-model tests:** navigation, commands, validation, diagnostics, state presentation.
- **Failure-path tests:** provider unavailable, partial subscription rollback, invalid variants, missing profiles, cycles, duplicate activation links, invalid controller maps.

Mocking rules:

- Mock or fake every external boundary: providers, file system, time, settings, dialogs, navigation, UI dispatcher, provider discovery, plugin catalog, CLI exit handling, runtime control endpoint, logging side effects, and output sinks.
- Prefer deterministic fakes for high-volume event streams.
- Prefer mocks for service interaction assertions and failure-path verification.
- Tests should assert diagnostics, not only boolean success/failure.
- Every bug in activation/runtime/mapping/migration should get a regression test.

#### Persistence and migration specification

Persistence must be versioned and testable.

Modern files:

- App settings: versioned JSON.
- Profiles/profile packs: versioned JSON.
- Runtime recordings/replay fixtures: versioned JSON.
- User variant packs and variant visuals: versioned XML by default, with a constrained schema.
- Legacy imports: legacy `context.xml` is read by migration tooling only.

Required persistence rules:

- Include `schemaVersion` in every modern persisted file.
- Store stable ids separately from display names.
- Preserve unknown fields where practical.
- Validate before saving and before activation.
- Back up legacy config before irreversible migration.
- Produce migration reports with warnings/errors that can be displayed in WPF and asserted in tests.
- Do not load arbitrary code, scripts, XAML, assemblies, or native libraries from user profile/variant files.

#### CLI and runtime-control specification

The CLI is a first-class automation surface.

Required commands:

```text
ucr activate <profile> [--fallback <profile>] [--no-additional] [--allow-additional-chain]
ucr deactivate
ucr status
ucr profiles list
ucr profiles validate [profile]
ucr variants validate <file-or-folder>
ucr replay <recording-or-fixture>
```

Required CLI behavior:

- Commands use the same DI registration and activation planner as WPF.
- Activation supports fallback only when the requested profile is missing.
- Status reports active profile, fallback use, additional profiles, controller groups, variants, providers, and diagnostics.
- Validate commands return non-zero exit codes on validation errors.
- CLI output should support human-readable text first; machine-readable JSON can be added with `--json`.

Runtime control endpoint:

- Optional for the first implementation.
- If implemented, use an abstraction first: `IRuntimeControlEndpoint`.
- Named pipes are the preferred Windows-local transport if maintainers need runtime IPC, but named pipes must live in an infrastructure adapter project, not core runtime.

#### Provider and IOWrapper absorption specification

IOWrapper becomes part of UCR but the provider boundary remains.

Required provider contracts:

- Discover input devices.
- Discover output devices.
- Report provider health.
- Report device capabilities.
- Report binding/control metadata.
- Subscribe/unsubscribe input controls.
- Subscribe/unsubscribe output devices.
- Set output state.
- Enter/exit bind detection mode, where supported.
- Report whether blocking/hiding input is supported for a control.

Provider rules:

- Provider adapters own native/API-specific dependencies.
- Provider adapters translate raw provider state into normalized runtime events.
- Provider adapters must not know WPF, Fluent UI, profile editor, or plugin UI concepts.
- Provider failures should produce typed errors/diagnostics, not unhandled exceptions across the runtime boundary.
- Provider subscriptions must be idempotent where possible and safely disposable.
- Provider-specific capabilities must be explicit. Do not assume every provider can block input, create virtual outputs, rumble, detect bind mode, or enumerate friendly names.

#### Acceptance criteria before legacy removal

Legacy removal is allowed only after maintainers accept parity evidence.

Minimum parity evidence:

- Legacy profile import covers representative real `context.xml` examples.
- Built-in plugin parity tests pass.
- Profile activation, additional activation, disabled chaining, enabled chaining, fallback, cycles, and duplicates are tested.
- Controller groups, overrides, shadow groups, and group-disabled-additional-loading are tested.
- Variant default/user packs, variant visuals metadata, and controller maps are tested.
- Runtime replay tests cover common mappings and at least one complex composed activation.
- Provider abstraction has fake-provider contract coverage and real-provider smoke tests where available.
- WPF app can activate/deactivate profiles, show diagnostics, edit profiles, manage variants, and preview inputs/outputs.
- CLI can activate, fallback, report status, validate, and replay.
- GitHub Actions runs default build/test workflow without requiring hardware, drivers, admin rights, or desktop interaction.

#### Decisions that are now fixed unless maintainers override them

- WPF is the desktop UI technology.
- Fluent UI for Windows is the design system; `WPF-UI` is the baseline implementation library.
- .NET Generic Host and Microsoft dependency injection are required.
- GitHub Actions and standard `dotnet` commands are the future build system.
- Modern tests use NUnit plus NSubstitute by default.
- Profiles compose through Additional profiles, not editable nesting.
- Additional profile chaining is disabled by default.
- Root profile has highest precedence in composed activations.
- Fallback profile is used only when the requested profile is missing.
- Variants are the mapping surface above DirectInput/XInput/raw providers.
- User variant packs use constrained XML by default.
- Runtime behavior must be replay-testable with input traces and expected output recordings.



## Repository layout

Current root-level areas:

- `UCR.sln` - legacy Visual Studio solution.
- `UCR/` - legacy Windows desktop application and WPF UI. The future UI should live in a modern WPF/Fluent UI app project rather than this legacy project.
- `UCR.Core/` - legacy core models, managers, binding/profile/plugin abstractions, and utilities.
- `UCR.Plugins/` - built-in legacy plugins, including filter and remapper plugins.
- `UCR.FileHandler/` - legacy file/unblock helper executable.
- `UCR.Tests/` - legacy NUnit test project.
- `build/` - legacy NUKE-based build project and build targets.
- `submodules/` - legacy external submodules, including IOWrapper. IOWrapper should be absorbed into the repo during the .NET 10 modernization rather than remaining a long-term submodule.
- `.github/`, `appveyor.yml`, `GitVersion.yml`, `build.ps1`, `build.cmd`, `build.sh` - legacy CI, versioning, and build entry points.

Generated or external directories such as `bin/`, `obj/`, `.tmp/`, `artifacts/`, `dependencies/`, `packages/`, `Providers/`, `Plugins/`, and submodule outputs should not be hand-edited.

## Build system policy

### Canonical future build system

GitHub Actions is the canonical build and verification system for the future .NET 10 version. The build contract for new work should be the standard .NET SDK CLI, invoked locally and from GitHub Actions.

Use these commands as the default contract for the modern solution:

```powershell
dotnet restore
dotnet build --configuration Release --no-restore
dotnet test --configuration Release --no-build
dotnet format --verify-no-changes
```

Guidelines for modern workflows:

- Put new workflows in `.github/workflows/`.
- Prefer clear workflow names such as `dotnet-ci.yml`, `release.yml`, and `dependency-audit.yml`.
- Use `actions/checkout` and `actions/setup-dotnet`.
- Pin the .NET SDK through `global.json` when a modern solution exists.
- Use `windows-latest` for WPF, HID, driver, `net10.0-windows`, installer, or packaging jobs.
- Use Linux/macOS runners only for platform-neutral libraries that actually build and test without Windows APIs.
- Keep CI steps readable. CI should call the same `dotnet` commands developers run locally.
- Do not hide the primary build behind custom scripts unless release packaging truly requires orchestration.
- Upload build artifacts from GitHub Actions for release candidates and tagged releases.
- Store signing certificates, package tokens, and release credentials only as GitHub Actions secrets or environments. Never commit them.

Recommended baseline workflow for the future .NET 10 solution:

```yaml
name: dotnet-ci

on:
  pull_request:
  push:
    branches: [ main, master, develop ]

jobs:
  build-test:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '10.0.x'
      - run: dotnet restore
      - run: dotnet build --configuration Release --no-restore
      - run: dotnet test --configuration Release --no-build
      - run: dotnet format --verify-no-changes
```

Adjust branch names, solution paths, and artifact steps when the actual modern solution layout exists.

### Legacy build system

The current repository uses a legacy NUKE build. Preserve it for maintaining the existing .NET Framework version, but do not treat NUKE as the build system for the .NET 10 rewrite.

Use these commands for the current legacy solution:

```powershell
git submodule update --init --recursive
.\build.ps1 InitProject
.\build.ps1 Clean
.\build.ps1 Test
.\build.ps1 Artifacts -Configuration Release
```

Notes:

- Prefer PowerShell on Windows for legacy build tasks.
- Visual Studio 2017-era assumptions may exist in the legacy solution.
- `appveyor.yml` is legacy CI configuration. Do not extend it for the future .NET 10 version unless explicitly asked.
- Some hardware/device-provider behavior may require Windows drivers or installed devices; do not make normal unit tests depend on those.
- If a legacy command fails because of missing local Windows tooling, report the exact command, failure, and what could not be verified.
- Do not migrate, upgrade, or delete the legacy NUKE build as incidental cleanup.

## Dependency modernization guidance

Dependency changes should be intentional, tested, and separated by legacy vs. modern scope.

### General dependency rules

- Do not update NuGet packages as incidental cleanup.
- Before changing dependencies, inspect all affected project files and run the most relevant build/test command.
- Prefer SDK-style `PackageReference` and `Directory.Packages.props` for the .NET 10 version.
- Use `packages.config` only for legacy projects that have not been intentionally migrated.
- Check current package versions at the time of work with NuGet, for example `dotnet list package --outdated`.
- For legacy packages, confirm compatibility with the existing target framework before proposing an upgrade.
- For modern packages, prefer packages that support `.NET 10`, `.NET 8+`, or `.NET Standard 2.0+` as appropriate.
- Document breaking changes and migration steps for any package major-version upgrade.

### Observed legacy dependency areas and modernization direction

| Area | Current legacy pattern | Modernization suggestion |
| --- | --- | --- |
| JSON | Newtonsoft.Json | Prefer `System.Text.Json` for new .NET 10 code unless Newtonsoft-specific behavior is required. If Newtonsoft is retained, isolate it and add serialization compatibility tests. |
| Logging | NLog direct usage | Prefer `Microsoft.Extensions.Logging` abstractions. Use NLog only behind the abstraction if needed. |
| HTTP | RestSharp | Prefer `HttpClient`/`IHttpClientFactory`; keep any REST client behind an adapter. |
| WPF theme | MaterialDesignThemes/Colors | Future UI decision is WPF with Fluent UI for Windows. Prefer a modern Fluent UI WPF control/theme stack for `UCR.App.Wpf`; do not upgrade legacy MaterialDesign packages unless doing a legacy-only maintenance fix with visual regression testing. |
| Tests | NUnit 3 console-era flow | Use current test SDK with `dotnet test`, whether choosing NUnit, xUnit, or MSTest. |
| Versioning | GitVersion 4-era packages | Re-decide release/versioning strategy for GitHub Actions before carrying GitVersion forward. |
| Build orchestration | NUKE build project targeting old .NET Core | Preserve for legacy only. Do not use as the .NET 10 build foundation. |
| NuGet CLI/MSBuild helper packages | Embedded/legacy build tooling | Prefer .NET SDK restore/pack/publish and GitHub Actions artifacts. |

For the legacy version, only upgrade dependencies to fix a confirmed bug, security issue, or required build break. Keep those changes isolated and verify with the legacy build. Do not modernize legacy dependencies as preparation for long-term support; the long-term target is replacement by the .NET 10 version and eventual legacy removal after parity.

## Coding guidelines

### General C# guidelines

- Keep changes small, focused, and directly tied to the requested task.
- Prefer readable, explicit code over clever abstractions.
- Avoid hidden side effects in constructors and property setters.
- Use clear names for devices, bindings, mappings, profiles, plugins, and providers.
- Validate inputs at boundaries, especially for profile/config loading and plugin/device discovery.
- Keep logging useful but avoid logging sensitive local paths, user profile data, tokens, or device identifiers unless needed for diagnostics.
- Do not add new runtime dependencies without a clear reason and a short explanation.

### Legacy C# guidelines

- Treat legacy code as reference and maintenance-only code.
- Respect the existing .NET Framework 4.5.2 and 4.6.1 constraints.
- Avoid introducing C# language features, APIs, or package versions that the legacy projects cannot compile.
- Preserve existing public APIs and plugin contracts unless the task explicitly requires a breaking change.
- Keep WPF/XAML changes consistent with the existing UI structure.
- Avoid broad formatting-only diffs in legacy files.
- Avoid converting old-style project files to SDK-style project files unless the task is specifically about migration.

### Modern .NET 10 guidelines

For new .NET 10 code:

- Enable nullable reference types.
- Prefer immutable models or narrowly scoped mutability for domain state.
- Use WPF for the desktop UI and keep WPF/Fluent UI dependencies in the app/UI layer only.
- Bootstrap the app through the .NET Generic Host; register services in one composition root instead of scattering static initialization through the app.
- Use constructor injection for windows, view models, services, runtime components, provider adapters, serializers, clocks, file systems, and hardware abstractions.
- Do not use service location from domain/runtime code. If an object needs a dependency, expose it through an interface and constructor parameter.
- Use `async`/`await` for I/O and long-running work; pass `CancellationToken` through service boundaries.
- Keep hardware polling, remapping loops, and event dispatch non-blocking where practical.
- Keep platform-neutral logic separate from Windows-only adapters.
- Add XML docs or Markdown docs for public plugin/provider extension points.
- Prefer central package/version management once the modern solution is established.
- Prefer clean, intentional APIs over copying legacy object models directly.
- When porting behavior, document whether the modern implementation is parity-equivalent, intentionally different, or not yet implemented.

## Architecture guidance

### Domain boundaries

Keep these concepts distinct:

- **Profiles**: user-facing mapping configurations and profile activation behavior. In the future model, profiles are independent activation units that may declare Additional profile activations.
- **Controller groups**: named sets of controller slots selected by a profile, including shadow groups and controller overrides.
- **Variants**: semantic input/output layouts layered above raw providers, used to make profiles portable and shareable.
- **Devices**: physical or virtual input/output devices and their metadata.
- **Bindings**: connections between variant/controller controls and mapped actions.
- **Mappings**: transformations from source inputs to desired outputs.
- **Plugins**: reusable units that extend remapping/filter/provider behavior.
- **Providers/adapters**: integrations with Windows APIs, drivers, IOWrapper, vJoy, ViGEm, Interception, Tobii, XInput, DirectInput, MIDI, or similar systems.

Domain logic should be testable without physical devices whenever possible.

## Legacy behavior model to preserve or deliberately redesign

The legacy code and wiki define several important UCR paradigms. Agents should preserve these semantics when the goal is parity, or document intentional differences when the .NET 10 version deliberately improves them.

### End-to-end data-flow paradigm

UCR is a pipeline:

1. Providers enumerate physical and virtual devices.
2. Profiles select input and output device configurations.
3. Mappings group one or more input bindings and one or more plugins.
4. Plugins transform normalized input values into output writes or filter-state changes.
5. Output providers send normalized output state to virtual or physical devices.

The hot path should stay small and predictable:

```text
provider input callback
  -> device binding callback
  -> mapping input cache update
  -> mapping plugin dispatch
  -> plugin transform
  -> output binding write
  -> provider output state
```

Do not put UI work, file I/O, network I/O, package restore, plugin discovery, device enumeration, or slow logging in the hot input-to-output path.

### IOWrapper absorption and backend ownership

IOWrapper should move into the UCR repo as internal backend code for the .NET 10 version. The goal is maintenance simplicity, not collapsing all abstractions into the UI.

Preserve these backend concepts:

- A provider abstraction for device APIs and drivers.
- Separate input and output capabilities; not all providers support both.
- Device enumeration returning provider reports, device reports, and binding trees.
- Input subscriptions identified by subscriber/profile identity, provider identity, device identity, and binding identity.
- Output subscriptions identified by subscriber/profile identity, provider identity, and device identity.
- Detection/bind mode as a separate provider mode from normal subscription mode.
- Output state writes routed through the active output subscription.
- Provider lifecycle and disposal.

Modernization direction:

- Prefer normal in-repo projects over submodules for IOWrapper code.
- Prefer explicit DI/composition over legacy MEF if it simplifies testing and packaging, but keep provider extensibility.
- Keep provider contracts small and versioned.
- Put platform-neutral provider abstractions in a neutral assembly and Windows driver/API implementations in Windows-specific assemblies.
- Treat device drivers, virtual bus clients, and native APIs as adapters around domain contracts.
- Preserve legacy normalized value semantics until a migration plan and compatibility tests justify a new representation.
- Add tests around backend subscription routing before replacing legacy IOWrapper behavior.

Suggested future ownership split:

```text
UCR.Core                  # profiles, mappings, filters, variants, controller groups, migration contracts
UCR.Runtime               # activation planner, runtime sessions, replayable pipeline
UCR.Plugins.Abstractions  # plugin contracts and metadata
UCR.Plugins.BuiltIn       # built-in remappers and filters
UCR.IO.Abstractions       # provider/device/binding/subscription contracts
UCR.IO                    # backend controller, provider registry, subscription routing
UCR.IO.Providers.Windows  # Interception, DirectInput, XInput, vJoy/virtual gamepad, MIDI, Tobii, etc.
UCR.App.Wpf               # WPF + Fluent UI shell, profile editor, diagnostics, app composition root
UCR.Cli                   # optional CLI activation/control front-end
```

### Device and binding model

The legacy device identity model is:

```text
ProviderName + DeviceHandle + DeviceNumber
```

`DeviceHandle` is provider-specific, often based on a USB VID/PID-like identifier. `DeviceNumber`/device instance disambiguates duplicate devices. Preserve stable duplicate-device ordering as much as the provider API allows.

Bindings are provider/device-specific and identified by:

```text
BindingType + Index + SubIndex
```

Legacy binding types are `Axis`, `Button`, and `POV`. Legacy binding categories are `Momentary`, `Event`, `Signed`, `Unsigned`, and `Delta`, which UCR maps into the UI/plugin categories `Momentary`, `Event`, `Range`, and `Delta`.

Agent rules:

- Do not key device identity only by display name.
- Do not assume device instance order is stable unless the provider guarantees it.
- Preserve binding trees/group labels because they are used for menus and user-facing binding names.
- Preserve `SubIndex` semantics for derived values such as POV directions.
- Honor provider-reported blockability; do not expose blocking for a binding unless the provider says it is blockable.
- Keep cached/disconnected-device behavior in mind when migrating profiles, because users may edit profiles while hardware is unplugged.

### Provider capability matrix and safety notes

Legacy providers and documented capabilities include:

| Provider/concept | Input | Output | Notes |
| --- | --- | --- | --- |
| Interception | keyboard, mouse | keyboard, mouse | Low-level; supports per-device bindings and blocking. Blocking is risky and must stay opt-in. |
| SharpDX DirectInput | non-Xbox joysticks/pads/wheels | no | Full stick-style input; no force feedback; cannot hide/block the device from games by itself. |
| vJoyInterfaceWrap | no | DirectInput-style virtual joystick | Requires driver/admin setup and manual vJoy device configuration; no force feedback in legacy docs. |
| SharpDX XInput | Xbox gamepads | no | Cannot hide/block or alter how other apps see a physical Xbox controller; no rumble support in legacy docs. |
| ViGEm | no | Xbox 360 / DualShock 4 virtual gamepads | Legacy docs describe virtual devices as created when an active profile needs them. Multiple-controller behavior was historically caveated and should be retested with the current driver stack. |
| Tobii Interaction | eye/head axes | no | Treat as optional hardware/API integration. |
| Titan One | controller state | controller state | Axis precision is limited in legacy docs; do not assume full 16-bit fidelity. |
| SpaceMouse | axes/buttons | no | 3Dconnexion-specific input provider. |
| MIDI | notes, pitch bend, control change | notes, pitch bend, control change | MIDI output can drive controls such as LEDs or motorized faders where supported. |

Tests and CI must not require admin-only drivers, real hardware, or machine-wide input blocking. Put such checks behind explicit manual or hardware-integration test categories.

### Legacy profile tree semantics

Legacy profiles form a tree. Child profiles inherit from parents. This section describes legacy/reference behavior for parity and migration; it is not the preferred profile model for the future .NET 10 version. The future model should use explicit Additional profile activations and controller groups.

Preserve or migrate these semantics deliberately for parity:

- Child profiles inherit parent device configurations, mappings, and filters.
- Activating a child profile activates applicable parent mappings first, then child mappings.
- A child mapping with the same title as a parent mapping overrides the parent mapping.
- The UI/user-facing path is a breadcrumb such as `Parent > Child > Grandchild`.
- Command-line profile activation searches comma-separated profile path segments breadth-first and case-insensitively.
- If a command-line path only partially matches, legacy behavior may activate the most specific match found; preserve or document any change.

For the .NET 10 version, make inheritance and overriding explicit in migration tests. Do not rely on incidental traversal order without documenting it. When replacing tree inheritance with Additional profile activations, document any behavior that is preserved, changed, or intentionally dropped.

### Runtime activation algorithm

Legacy profile activation is subscription-state based. For parity, model activation as building a new state, subscribing it, then swapping it in.

Expected phases:

1. Refresh backend devices unless explicitly skipped.
2. Build a new subscription state for the requested active profile.
3. Recursively include parent profiles before child profile data.
4. Add output device configuration subscriptions.
5. Add mapping subscriptions and mark parent mappings overridden by child mappings of the same name.
6. Create shadow mapping subscriptions for eligible mappings.
7. Build the runtime filter dictionary with all known filter names initialized inactive.
8. Subscribe outputs, including shadow outputs.
9. Prepare mappings by building the input cache, callback multiplexers, filter-state reference, and runtime plugin links.
10. Subscribe input bindings.
11. If activation succeeds, deactivate the previous active state.
12. Initialize plugin cache values, call plugin activation hooks, set active profile, and notify listeners.

Important constraints:

- Activation should be transactional. If any subscription fails, unsubscribe anything already subscribed for the failed state.
- Deactivation should call plugin deactivation hooks and unsubscribe input and output subscriptions, including shadow devices.
- Re-activating the same profile should be idempotent.
- The new implementation should avoid the legacy risk of partial activation with stale output devices.
- Log activation failures with provider/device/binding context, but avoid leaking sensitive local paths or unnecessary device identifiers.

### Mapping execution model

A mapping owns input bindings, an input cache, and one or more plugins. Each bound input updates one slot in the mapping cache. When any input changes, all plugins in the mapping receive the full current input cache unless filtered out.

Rules to preserve:

- The first plugin added to a mapping determines the mapping's input binding shape.
- Additional plugins in the same mapping must have compatible input categories.
- Multiple plugins in one mapping can reuse the same input binding set to emit different outputs under different filters.
- Plugins should receive normalized values only; provider-specific raw data belongs behind provider adapters.
- Plugin output writes should update the output binding's current value before forwarding to the provider sink.
- Mapping/plugin code may be stateful, but state must be resettable on activation/deactivation.

### Filter semantics

Filters are named runtime booleans used to control whether plugins receive input.

Legacy semantics:

- Filter names are user-entered text and should be treated case-insensitively.
- Filters are inherited from parent profiles.
- A plugin with no filters is unfiltered and should receive input.
- A plugin with filters should receive input only when every assigned filter condition is satisfied.
- Inverted/negative filters mean the plugin expects that filter to be inactive.
- Filter plugins mutate runtime filter state; normal remapper plugins consume it.
- Shadow mappings must have isolated filter names so each shadow clone can maintain independent modifier state.

Preserve `Button to Filter` and `Axis to Filter` behavior as distinct plugin patterns:

- Button filter plugins can set a filter active, inactive, toggle it, or leave it unchanged on button down/up.
- Axis filter plugins compare the axis against configured lower/upper percentage bounds and mutate filter state on range entry/exit.

### Shadow device semantics

Shadow devices duplicate one mapping configuration across multiple identical or equivalent device pairs.

Preserve these user-visible rules:

- A primary input maps to the primary output.
- Shadow input `N` maps to shadow output `N`.
- Ordering matters; changing input or output shadow order changes which player/device maps to which virtual device.
- If input and output shadow counts differ, behavior must be deterministic and tested.
- Shadow clones should not pollute the user's persisted mappings as independent user-authored mappings unless explicitly redesigned.
- Filter state for shadow clones must be isolated, commonly by deriving shadow-specific filter names from the original filter name and shadow index.

### Plugin metadata and lifecycle

Legacy plugins are discovered from plugin folders and described through attributes. The modern version may use a different discovery mechanism, but should preserve the metadata model:

- Plugin display name, group, description, and disabled state.
- Ordered input definitions with names and categories.
- Ordered output definitions with names, categories, and optional groups.
- GUI/settings properties with display names, order, group, and validation.
- Optional settings/output grouping for UI layout.

Plugin lifecycle hooks to preserve conceptually:

```text
InitializeCacheValues()
OnActivate()
Update(short[] values)
OnPropertyChanged()
OnDeactivate()
```

Modern plugin contracts may rename these methods, but keep the same responsibilities:

- Precompute expensive settings-derived values before hot-path updates.
- Initialize outputs when a plugin/profile activates if required.
- Keep update methods fast and deterministic.
- Validate settings before runtime activation.
- Clean up subscriptions/state on deactivation.

### Normalized value model

Legacy UCR/IOWrapper normalizes input and output values so plugins do not need to know provider-specific ranges.

Legacy ranges:

- Axis/range values use signed 16-bit values: `short.MinValue` to `short.MaxValue` (`-32768..32767`).
- Momentary button values use `0` for released and `1` for pressed.
- Delta values represent continuous relative movement, such as mouse or eye-tracker deltas.
- Events represent stateless impulses, such as a mouse wheel tick.

For .NET 10, consider wrapping raw `short` values in typed value objects, but do not change persisted or runtime semantics without migration tests.

### Axis and remapping algorithms

The legacy algorithms below are important for parity tests.

General axis helpers:

- Invert maps min to max, max to min, zero to zero, and otherwise multiplies by `-1`.
- Clamp clamps integer values into signed 16-bit axis range.
- Percentage conversion maps `-100..100` to the signed 16-bit range, with `-100` explicitly returning `short.MinValue`.
- Split-axis converts one centered axis into high/low half-axis outputs stretched to full range.
- Safe absolute value must handle `short.MinValue` without overflow.

Dead-zone helpers:

- Linear dead zone clamps percentage to `0..100`, returns zero inside the cutoff, and rescales the remaining range back to full signed-axis magnitude.
- Anti-dead-zone leaves zero at zero; non-zero values jump to the configured anti-dead-zone start and scale the remainder toward max.
- Circular dead zone treats two axes as a vector, zeros vectors inside the radius, then rescales vector length from radius-to-max back to full range.
- Sensitivity can be linear scaling or a nonlinear curve; preserve legacy curve behavior with golden tests before attempting to improve it.

Built-in remapper patterns:

- `Axis to Axis`: optional invert, dead zone, anti-dead-zone, sensitivity, then output.
- `Axes to Axes`: two-axis joystick transform with optional circular or per-axis dead zone, sensitivity, clamping, and per-axis invert.
- `Axis to Button`: axis sign after dead zone drives high/low button outputs; neutral releases both.
- `Button to Axis`: released and pressed button states map to configured axis percentages; can initialize axis on activation.
- `Buttons to Axis`: two buttons drive one axis; both or neither pressed returns neutral.
- `Axis Splitter`: one axis becomes high and low outputs, with optional dead zone and per-side inversion.
- `Axis Merger`: two axes merge by average, greatest absolute magnitude, or clamped sum, then optional dead zone/sensitivity.
- `Button to Event`: button edge should emit event-style output only on the intended transition.
- `Axis Initializer`: activation-time output initialization is a first-class behavior, not a UI-only convenience.

When porting these algorithms, add golden tests around representative values: min, max, zero, just inside/outside dead zone, sign changes, both-buttons-pressed, shadow clones, and filter transitions.

### Bind-mode algorithm

Legacy bind mode asks all devices in the relevant profile device configuration list to enter detection mode for a short window.

Behavior to preserve or consciously redesign:

- Bind mode is typed: a binding only accepts input matching the required category.
- Event and delta inputs can be accepted immediately.
- Momentary inputs require a non-zero value.
- Range inputs require a deliberate axis displacement rather than idle noise.
- Once a valid input is detected, UCR records the device configuration GUID plus binding type/index/sub-index, then exits bind mode.
- Providers must be returned to normal subscription mode when bind mode ends or is cancelled.

Tests should cover cancellation/timeouts and provider cleanup even when no input is detected.

### Profile/config persistence and migration

Legacy UCR stores the main user configuration in `context.xml` and uses XML serialization with plugin concrete types. Device cache files are JSON files under provider-specific cache folders.

Migration rules:

- Treat `context.xml` as the primary legacy profile source.
- Maintain migration tests with real or representative legacy profile XML.
- Do not require all legacy plugin assemblies to be present merely to inspect or migrate profile metadata if a safer migration parser can be written.
- Preserve unknown plugin/config data where practical so users do not lose profiles when optional plugins are unavailable.
- Device caches are helpful for editing disconnected devices, but should not be required for runtime activation when devices are present.
- Migrate legacy nested profile behavior into Additional profile activations and controller groups only with tests that show equivalent or intentionally changed behavior.
- Preserve enough legacy identity data to let users recover profile/controller relationships after migration.
- When introducing variants, include migration fixtures for default XInput/DirectInput naming, user-defined labels, and controller-map overrides.
- Any irreversible migration must be explicit, documented, and backed up.

### Architecture risks to avoid during modernization

- Do not merge UI, profile domain, plugin execution, and provider I/O into one project just because IOWrapper is moving into the repo.
- Do not make the new plugin system depend on WPF controls or view models.
- Do not make tests depend on physical controllers, virtual bus drivers, administrator rights, or global keyboard/mouse hooks.
- Do not replace the normalized value model without compatibility tests.
- Do not silently change profile inheritance, mapping override, filter, or shadow-device behavior.
- Do not keep legacy MEF/provider folder layout solely for nostalgia; keep it only if it remains the best packaging model.
- Do not carry over legacy bugs accidentally. Capture existing behavior with tests first, then decide whether the .NET 10 version should preserve or fix each quirk.

## Testing expectations

- Add or update tests for behavior changes.
- Prefer unit tests for core mapping, binding, profile, plugin, provider-routing, runtime, activation planning, variant/controller-map, and migration logic.
- All modern services must be mockable or replaceable with fakes through DI.
- Avoid tests that require real controllers, keyboards, mice, eye trackers, vJoy, ViGEm, Interception, administrator privileges, desktop interaction, or global input hooks.
- Use fakes/mocks/adapters for hardware, driver integrations, clocks/time providers, UI dispatchers, file systems, settings, CLI exits, runtime transports, plugin discovery, and provider registries.
- Add runtime replay tests that feed recorded timestamped input events through fake providers and assert expected output recordings.
- For bug fixes, add a regression test when practical.
- For .NET 10 porting work, add tests that capture the legacy behavior being matched or intentionally changed.
- Add golden tests for axis algorithms, filters, mapping overrides, shadow devices, bind mode, Additional profile activations, controller groups, variants/controller maps, fallback CLI activation, WPF view models, and `context.xml` migration before declaring parity.
- WPF UI tests should focus on view models, services, navigation behavior, and diagnostics models. Do not require normal CI to launch real windows unless a dedicated UI test category is explicitly configured.
- When tests cannot be run locally, explain why and list the commands that should be run by someone with the right environment.

Legacy tests currently live under `UCR.Tests/`. Future .NET 10 tests should live under `tests/` or a similarly clear modern test structure. Keep all modern tests runnable through `dotnet test`.

## Documentation expectations

Update documentation when behavior, setup, architecture, plugin contracts, CI/build behavior, dependencies, provider integrations, or migration behavior changes.

Useful places to update include:

- `README.md`
- `CHANGELOG.md`
- `.github/workflows/*.yml`
- wiki-linked documentation, if the task includes docs work
- new modernization docs such as `docs/architecture.md`, `docs/migration.md`, `docs/build.md`, `docs/plugins.md`, `docs/providers.md`, or `docs/parity.md`

Do not rewrite user-facing docs as part of unrelated code changes.

## Git and pull request guidance

Follow the repository's existing contribution style unless maintainers give newer instructions.

- Use `feature/` branches for new functionality.
- Use `hotfix/` branches for fixes.
- Do not target release branches directly unless instructed.
- Use imperative, present-tense commit messages.
- Keep the first commit-message line concise.
- Reference relevant issues or pull requests when known.
- Update `CHANGELOG.md` for user-visible changes when appropriate.

## Safety and boundaries

- Never commit secrets, credentials, API keys, certificates, signing keys, or personal tokens.
- If existing source appears to contain a credential, do not copy it into new files or documentation.
- Do not install, remove, or configure Windows drivers unless the task explicitly asks for it.
- Do not enable low-level keyboard/mouse blocking by default.
- Do not run destructive commands such as deleting user profiles, uninstalling drivers, or wiping generated device configuration.
- Do not edit binary assets unless the task is specifically about those assets.
- Do not edit generated outputs by hand.
- Do not update submodules, NuGet packages, build tools, or CI providers as incidental cleanup.
- Do not mix legacy-retargeting work with unrelated feature work.
- Do not remove legacy code, projects, tests, build scripts, or CI files before accepted .NET 10 feature parity and an explicit removal task.

## Agent workflow

Before changing code:

1. Identify whether the task is legacy maintenance, legacy behavior research, .NET 10 modernization, WPF/Fluent UI work, dependency-injection/service design, runtime replay testing, IOWrapper absorption, profile/variant/controller-group design, parity tracking, or legacy removal.
2. Inspect the relevant project files and existing tests.
3. Inspect dependency impact if the task touches packages, build, serialization, logging, HTTP, WPF/Fluent UI, dependency injection, plugin contracts, provider integrations, runtime replay, or profile migration.
4. Prefer the smallest safe change.
5. For .NET 10 porting work, identify the legacy behavior being matched and whether parity tests/docs are needed.
6. Call out assumptions when hardware, drivers, or missing tooling affect verification.

Before finishing:

1. Run the most relevant build/test commands available for the touched area.
2. Inspect the diff for accidental formatting, generated-file, dependency, or CI changes.
3. Update docs/tests if behavior changed.
4. For modernization work, update parity notes if a legacy feature was implemented, intentionally changed, or deferred.
5. Summarize changed files, verification performed, and any remaining risks.

Final responses should include:

- What changed.
- Why it changed.
- Tests or checks run.
- Any commands that could not be run and why.
- Any compatibility or migration concerns.

## When to ask before proceeding

Ask for maintainer direction before:

- Moving legacy code into a new folder layout.
- Absorbing IOWrapper into a specific new namespace/project layout if no modernization plan exists.
- Declaring the .NET 10 version feature-complete or parity-equivalent with the legacy version.
- Removing legacy code, legacy tests, legacy build scripts, or legacy CI configuration.
- Retargeting legacy projects from .NET Framework to .NET 10.
- Changing plugin, provider, binding, filter, shadow-device, profile activation, controller-group, variant, controller-map, or profile serialization contracts.
- Replacing the decided WPF + Fluent UI direction, build system, CI provider, dependency-injection model, or driver stack.
- Adding new runtime dependencies or external services.
- Removing support for existing devices, providers, plugins, or profile formats.
