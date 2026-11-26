# AGENTS.md — Building UCP3 extensions

UCP3 = Lua 5.4 modding framework for Stronghold Crusader (SHC/SHCE 1.41, 32-bit x86).
Framework source: `UnofficialCrusaderPatch3/content/ucp/code/*.lua` (`core.lua`, `main.lua`, `extensions/*.lua`).
Good reference extensions: `gynt/ucp_startResources` (simple module), `UnofficialCrusaderPatch/extension-files` (public API),
`TheRedDaemon/ucp_gmResourceModifier` (DLL module), `Krarilotus/extension-Aggressive-AI-Behaviour(-Applied)` (content plugin + config plugin).
See the [API owners and pins](https://github.com/UnofficialCrusaderPatch/ucp-agent-guide/blob/main/docs/where-to-find.md) and [past mistakes](https://github.com/UnofficialCrusaderPatch/ucp-agent-guide/blob/main/docs/mistakes.md).

## Extension types
- **module** — may patch game memory/code, load DLLs. Has full `_G` (`core`, `utils`, `hooks`, `io`, ...). Lives in `ucp/modules/<name>-<ver>/`; distributed as `<name>-<ver>.zip` (trusted via the Store's security catalog or an adjacent `.sig`; dev builds skip trust check).
- **plugin** — content + config only, cannot touch memory. Lives unzipped in `ucp/plugins/<name>-<ver>/` (store ships a zip the launcher extracts); game assets go in `resources/` (e.g. `resources/ai`, `resources/maps`). `init.lua` optional.

Folder name MUST be `<name>-<semver>`; name `[A-Za-z0-9-]+`; version must equal `version` in definition.yml.
Path alias: `ucp/modules/<name>/` and `ucp/plugins/<name>/` resolve to the versioned folder — use these in configs (`io.resolveAliasedPath(p)` to expand).

## Files
```
definition.yml        required
init.lua              module: required; plugin: optional
options.yml           user-facing options + defaults (GUI)
config.yml            demands on other extensions' options
locale/en.yml         flat key: text for '{{key}}' in options.yml (also tags.<tag>, family.<id>)
locale/description-en.md   GUI description (add -de, -fr, ... as needed)
*.lua, subfolders, *.dll (modules only), resources/ (plugins)
```

### definition.yml
```yaml
name: myExtension            # == folder name prefix
display-name: "My Extension"
author: you
version: 1.0.0               # semver, == folder suffix
type: module                 # or plugin
meta: { version: 1.0.0 }     # file-format version, keep 1.0.0
dependencies:                # name: semver range (^1.0.0, ">=1.0.2", '>=1.0.17 <2.0.0')
  framework: ^3.0.0          # reserved: UCP3 framework
  frontend: ^1.0.0           # reserved: GUI launcher
  files: ">=1.3.0"           # other extensions you call or configure
tags: [ai]                   # optional
```
Declare every extension you call or configure: dependencies decide load order.

### options.yml
URLs start with your extension name; the rest becomes the path into your `config` table.
```yaml
meta: { version: 1.0.0 }
options:
- name: grp
  display: GroupBox          # seen: GroupBox, Number, Switch, Slider, Choice, CustomMenu
  header: '{{my_header}}'    # locale key
  text: '{{my_descr}}'
  children:
  - url: myExtension.feature.enabled
    display: Switch
    text: Enable feature
    contents: { type: boolean, value: false }
  - url: myExtension.feature.amount
    display: Number          # Slider adds step/decimals
    text: Amount
    contents: { type: number, min: 0, max: 1000, value: 5 }   # value = default; keep vanilla
  - url: myExtension.feature.mode
    display: Choice
    contents:
      type: choice
      choices: [{name: a, text: "A"}, {name: b, text: "B"}]
      value: a
```
Modules should be unopinionated: many toggles, vanilla defaults. Presets and their families belong in plugins.

### config.yml (typically plugins)
```yaml
meta: { version: 1.0.0 }
config-sparse:
  modules:
    aiSwapper:               # target extension
      config:
        menu:                # option path without the extension prefix
          contents:
            suggested-value: ...      # new default, user can override
            # required-value locks the value, including false. The GUI reports conflicting locks.
  plugins: {}
```
Paths to your own files: `ucp/plugins/<name>/resources/...` (unversioned alias).

## init.lua contract
```lua
local feature = require("feature")        -- ./feature.lua or ./feature/init.lua

local exports = {}

function exports:enable(config)            -- called at startup, in load order
  -- config = THIS extension's options, prefix stripped, contents already resolved to plain values:
  --   myExtension.feature.amount -> config.feature.amount
  -- Values can be nil: always guard.
  if config.feature and config.feature.enabled then feature.apply(config.feature.amount) end
end

function exports:disable(config) end      -- not called by the framework; keep as no-op

function exports:doThing(x) return x * 2 end   -- API for other extensions

return exports, { public = { "doThing" } }      -- public = what plugins may call
```
Startup order: all extensions' `init.lua` loaded (in load order) → all `enable` called (in load order) → game loading screen → `afterInit` hooks → game main loop.

## Sandbox rules (enforced)
- **No global assignment**: `x = 1` errors. Use `local` everything; store state on `self`/locals.
- `require("a.b")` is relative to your extension root; dots = folders; never add `.lua` (errors). Results are cached per extension. Framework libs are already globals, not requireable.
- `require("name.dll")` (modules only) loads `name.dll` from your root and calls its `luaopen_name`; returns that table.
- `print(msg)` / `log(LEVEL, msg)` auto-prefix your name. Levels: `FATAL ERROR WARNING INFO DEBUG VERBOSE` (level names exist in modules; plugins use `print`).
- Plugin env contains ONLY: `modules` (public members), `plugins`, `utils`, `table`, `string`, `type`, `pairs`, `ipairs`, `log`, `print`, `require`, `userdata` (3.0.8). No `core`, `io`, `hooks`, `math`, `tostring`, `pcall`, ... If you need any, write a module.
- Module env = `_G` + `modules`/`plugins` (full proxies).

## Calling other extensions
```lua
modules.files:overrideFileWith("gfx/foo.tgx", "ucp/plugins/myPlugin/resources/foo.tgx")   -- always use ':'
```
- Proxies force `self` to the target; `enable`/`disable`/`init` are blocked.
- Returned tables become **read-only** proxies; **only the first return value** is passed through — return a table for multiple values.
- Plugins see only names listed in the target's `public`.
- Assigning fields on `modules.x` errors.

## hooks
```lua
hooks.registerHookCallback("afterInit", function() ... end)   -- only hook; runs just before main loop
```
Use for anything that needs game objects constructed (UI menus, AI data, `userdata`). Errors are caught and logged.

## core API (modules)
Memory: `readInteger/readSmallInteger/readByte(addr)`, `readBytes(addr,n)`, `readString(addr[,len])`,
`writeInteger/writeSmallInteger/writeByte/writeBytes/writeString(addr,v)` (data memory),
`allocate(size[,zero=true])`, `deallocate(addr)`, `copyMemory(dst,src,n)`, `setMemory(dst,v,n)`,
`registerString(s)` (permanent, leaks), `CString(s)` (`.address`, freed on GC), `allocateGarbageCollectedObject(size[, zero])` (3.0.8; `.address`, freed on GC).

Code memory: `writeCode(addr, codeTable[, compile=true])`, `writeCodeBytes/writeCodeByte/writeCodeInteger/writeCodeSmallInteger`,
`allocateCode(nBytes | codeTable)`, `deallocateCode(addr)`.

Finding addresses:
- `core.AOBScan("8b 44 24 ? 50"[, start[, stop]])` → first match (no uniqueness check); **throws** if not found; results cached. Pass `start` with `stop`. In 3.0.7, check the returned address against your range; in 3.0.8, `stop` bounds the match's start. For uniqueness use `core.AOBScanUnique(pattern)` (3.0.8) → address, or `nil, reason`.
- `utils.AOBExtract("@(E8 ? ? ? ?) 8B 0D I(? ? ? ?)")` → addr, captures. `@()` = rel call/jmp target made absolute (the opcode goes inside `@()`), `I()` = int32, `()` = bytes.
- Use AOBs, never hard-coded addresses (SHC vs SHCE, versions differ). Read pointers out of instructions: `core.readInteger(core.AOBScan(aob) + offset)`.

Hooking:
- `orig = core.hookCode(luaFn, addr, argCount, CallingConvention.CDECL|THISCALL, hookSize)` — luaFn gets ints (`this` first for THISCALL); call `orig(...)` for original. stdcall not implemented.
- `core.detourCode(function(r) ... return r end, addr, size)` — `r.EAX..r.ESI`; **must return r**; size ≥5 and on instruction boundary.
- `fn = core.exposeCode(addr, argCount, cc)` — call game function from Lua; returns int (1/0 for bools; `TRUE`/`FALSE` globals).
- `core.insertCode(addr, patchSize, codeTable[, returnTo][, "before"|"after"])` — jmp to new code, then back; "before"/"after" re-includes overwritten bytes (copied verbatim: relative jumps/calls in them are not relocated).
- `core.assemble(fasmSrc, {SYM=0x...}, origin)` → bytes; `core.allocateAssembly(src, map)` → address; `core.AssemblyLambda(src, map)` for inline use.
- `utils.createLuaFunctionWrapper(fn)` → native address that detours into `fn(registers)`.

Code tables (compiled by `writeCode`/`insertCode`): byte `0x00–0xFF` → 1 byte; **any other number (incl. negatives, ≥0x100) → 4 bytes LE**; nested tables flattened; strings = labels;
helpers `core.callTo(a)`, `jmpTo(a)`, `jeTo(a)`, `relTo(a|label[,off])`, `byteRelTo`, `shortRelTo`, `relToCall(a)`; `core.itob/stob` for explicit bytes. Use `writeCodeBytes` to write raw bytes without compiling.

DLLs: build 32-bit, link the framework's `lua.dll` (5.4); export `luaopen_<name>` returning function pointers/addresses; wire them up in `enable` (e.g. `core.writeCode(callSite, {core.callTo(dll.funcAddr)})`). `core.openLibraryHandle(path)` → `:require(mod)`, `:getProcAddress(fn)`.

## Files & data
- `io.open` is restricted to the game dir; use `ucp/...` paths. Installed module and plugin files are read-only: write through `userdata`. Modules can `io.mkdir`, `io.remove` (3.0.8).
- Per-user persistent storage: `local ud = userdata.interface()` (3.0.8; nil before afterInit) → `ud:open(rel, mode)`, `ud:mkdir`, `ud:remove`, `ud:rename(from, to)`, `ud.version`, `ud:setVersion("x.y.z")`.
- Game file overrides / extra resources: depend on the `files` module (`overrideFileWith`, `registerOverrideFunction`, `registerFileSource`, `registerExtraMapDirectory`).
- Globals available in modules: `json`, `yaml`, `sha`, `utils` (`table.find/keys/values/length/update/dump`, `string.split`, `utils.pack/unpack`), `data.version.isExtreme()`, `data.version.getGameVersionMajor()`, `data.version.getGameVersionMinor()`, `data.version.getGameLanguage()` (nil before afterInit).

## GitHub repo layout & Extension Store
Store = `UnofficialCrusaderPatch/UCP3-extensions-store`, one branch per framework version (e.g. `3.0.7`; `recipe.yml` `framework.version: =3.0.7` must match the branch).
`scripts/build-store.ps1` reads `recipe.yml`, fetches each repo at a pinned commit, builds/zips/signs, and publishes `store.yml` + zips as a GitHub release.

Repo layout: one extension per repo (convention `extension-<name>`), extension files at the **repo root** (folder name is not used — name/version come from `definition.yml`):
```
definition.yml  init.lua  options.yml  config.yml  *.lua
locale/en.yml   locale/description-en.md   (+ de fr ru hu tr ch es fa)
resources/      (plugins)
files.xml | files.yml   (modules, optional: what goes into the zip)
build.ps1 | meson.build | *.sln   (modules with native code, optional)
README.md, LICENSE, .gitignore
```
Monorepo: put each extension in a subfolder and set `location: <subfolder>` in the recipe (sibling folders are dropped, but repo-root files can still end up in the ZIP: inspect it).

What gets packaged:
- **Plugin**: entire extension folder copied as-is into `<name>-<ver>.zip` → keep dev junk (tests, src assets, scripts) out of the repo/subfolder, or use a `location` subfolder.
- **Module**: if `files.xml`/`files.yml` exists only listed files are copied (`files.xml` wins if both exist); otherwise the whole folder. Native code is compiled first (`build.ps1` > `meson.build` > `*.sln`, Release config, against the UCP3 NuGet package), then zipped and signed.
```xml
<module>
  <project path="src/my.vcxproj"/>
  <files>                         <!-- add build="Debug" variant if needed -->
    <file src="src/$(Configuration)/myModule.dll" />
    <file src="*.lua" target="." />
    <file src="*.yml" target="." />
    <file src="locale" target="." />
  </files>
</module>
```
```yaml
# files.yml equivalent
meta: { version: 1.0.0 }
files:                     # or files-Release / files-Debug
- src: 'src/$(Configuration)/myModule.dll'
- src: '*.lua'
- src: '*.yml'
- src: locale
  target: .
```

Store-enforced rules:
- `definition.yml` at the pinned commit: `version` must equal the recipe's version; `type` must be `module` or `plugin` (or overridden in the recipe); `dependencies` is optional; if present it **must be a map** (old list format aborts the build).
- Every change needs a **version bump**: binaries are cached by name+version from earlier releases (same-version rebuild only with `rebuild-on-source-change: true`, maintainer-approved). A recipe version lower than an already-released one fails.
- The `github-sha` must exist on `github-tag` (branch or tag); commit pushed before the PR.
- Descriptions are linked from `raw.githubusercontent.com/<repo>/<sha>/[location/]locale/description-<lang>.md`; `en` also becomes the default. Missing languages are skipped.
- Store capabilities are derived from the zip: `files` (has `resources/`), `code` (has `init.lua`), `configuration` (config.yml `config-sparse.modules|plugins` contains a `contents`), `options` (non-empty `options:` list).
- Tags: `tags: [ai, maps]` → labels from `locale/<lang>.yml` keys `tags.<lowercase-id>`.
- Families (base content + "applied" config variants): the root lists `family: [{name: <id>, root: true}]`, other members `family: [{name: <id>}]` (exact lowercase keys `name`/`root`; case-sensitive ids, distinct within a package even ignoring case). Each family needs exactly one root package (any of its versions may declare `root: true`) and ≥2 packages across the store. Labels: `family.<id>` and `family.<id>.configuration` in locale files.

Recipe entry (PR to the store repo, in `recipe.yml` under `extensions.list`):
```yaml
    - definition:
        name: myExtension
        version: 1.0.0
        type: plugin              # or module
      contents:
        source:
          method: github
          url: owner/extension-myExtension   # owner/repo, no https://
          github-tag: main                    # branch or tag containing the sha
          github-sha: <full 40-char commit>
          # location: subfolder               # monorepo only
```
Add a line to `docs/changelog/<branch>.md`. Release flow: tag/push your repo → PR recipe bump with new version + sha → maintainers build & sign.

## Checklist
1. Folder `<name>-<ver>`, matching `definition.yml` name/version/type, dependencies listed.
2. `init.lua` returns table with `enable` and `disable`; no globals; `require` without `.lua`.
3. Option URLs prefixed with extension name; vanilla defaults; nil-guard config in `enable`.
4. AOB everything; detours return registers; hook sizes cover whole instructions.
5. Public API: functions use `:`, listed in `public`, return a single value.
6. Locale keys exist in `locale/en.yml`; `locale/description-en.md` present.
7. Repo root = extension root (or `location`), clean of dev files; version bumped; commit pushed; recipe entry pins that sha.

## Rules and reasons
- Identify setups by extension versions and effective settings; never hash package files. Reason: Store versions are authoritative, and the framework already verifies module ZIPs at load.
- Resolve each game-code binding once at startup; if it is missing or ambiguous, log `WARNING` and leave that optional feature off. Reason: a first match is not proof of the right site, and one missing site should not stop unrelated features.
- Use `data.version.getGameLanguage()` (modules, after `afterInit`) for in-game text, not an extension language option. Reason: one UCP setting keeps all texts in the same language.
- Reuse the framework and owner-module APIs listed in `docs/where-to-find.md`. Reason: duplicated helpers drift and conflict.
- Keep launcher texts short and about the whole extension. Reason: players read current behaviour, not a changelog ([extension-aiSwapper#10](https://github.com/UnofficialCrusaderPatch/extension-aiSwapper/issues/10#issuecomment-2874654884)).