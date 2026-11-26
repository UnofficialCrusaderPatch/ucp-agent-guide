# Common extension mistakes

## Duplicating framework memory helpers
- **What went wrong:** Early `value_sites.lua` in [CIO61/rebalancer#15](https://github.com/CIO61/rebalancer/pull/15) rebuilt width-aware reads on top of byte reads.
- **Do instead:** Use `core.readByte`, `core.readSmallInteger`, `core.readInteger`, and the matching writers at the target framework revision.
- **Source:** [CIO61/rebalancer#15](https://github.com/CIO61/rebalancer/pull/15), `value_sites.lua:19–54@42b2349` → `:5–9@1789189`.

## Writing runtime files under `ucp/` or content folders
- **What went wrong:** Early [gynt/ucp-extension-custom-skirmish-trails#10](https://github.com/gynt/ucp-extension-custom-skirmish-trails/pull/10) used `winfiles` to write progress under the game installation's `ucp/` directory and synced back to paths derived from trail mission files.
- **Do instead:** Write persistent runtime output through `userdata.interface()` (3.0.8) and leave package and content folders read-only.
- **Source:** [gynt/ucp-extension-custom-skirmish-trails#10 review](https://github.com/gynt/ucp-extension-custom-skirmish-trails/pull/10#discussion_r4702038914), `init.lua:202–206@04e53bf`, `customskirmishtrails/progress.lua:4,76–89@04e53bf`, `customskirmishtrails/sync.lua:232–238,274–299@04e53bf` → `customskirmishtrails/storage.lua:13–61@d6b3cae`.

## Hashing extension files at admission
- **What went wrong:** [gynt/ucp-extension-protocol#3](https://github.com/gynt/ucp-extension-protocol/pull/3) walked installed extension ZIPs, members, and configured file contents to hash them for admission checks.
- **Do instead:** Base compatibility and admission on active extension versions and effective setup identity (`data.version.getDigest()`; canonicalized with framework version and load order in 3.0.8).
- **Source:** [gynt/ucp-extension-protocol#3](https://github.com/gynt/ucp-extension-protocol/pull/3), `admission/content.lua:59–112@b2acd33`; [extension-files#10 version guidance](https://github.com/UnofficialCrusaderPatch/extension-files/issues/10#issuecomment-5654507205).

## Mis-resolving `@()` captures after pattern start in `utils.AOBExtract`
- **What went wrong:** In framework `v3.0.7`, `@()` reads the relative displacement at the capture offset but adds it to the match start instead of the capture position, resolving the wrong address when bytes precede the captured opcode.
- **Do instead:** Use the fixed extractor in 3.0.8, or put `@()` at byte 0 of the pattern (or add the capture's byte offset to the returned address) when targeting 3.0.7.
- **Source:** `content/ucp/code/utils.lua:423–433@77c6acc` → `:437–446@a67270a`.

## Putting technical format details in launcher descriptions
- **What went wrong:** A module's `locale/description-en.md` included a version-specific file reference instead of summarizing the extension's overall use.
- **Do instead:** Keep `locale/description-<lang>.md` short and scoped to the whole module or plugin, and move technical file formats to documentation files.
- **Source:** [gynt/ucp-extension-custom-skirmish-trails#10](https://github.com/gynt/ucp-extension-custom-skirmish-trails/pull/10), `locale/description-en.md:97@afaff5b` → `docs/trail-files.md:1@5ed89f8`.

## Logging with `WARN` instead of `WARNING`
- **What went wrong:** Passing undefined `WARN` passes `nil` as the log level instead of logging a warning.
- **Do instead:** Use the framework's defined level `WARNING` (`FATAL`, `ERROR`, `WARNING`, `INFO`, `DEBUG`, `VERBOSE`).
- **Source:** [extension-aiSwapper#6 comment](https://github.com/UnofficialCrusaderPatch/extension-aiSwapper/issues/6#issuecomment-2466963348), `content/ucp/code/logging.lua:3–17@a67270a`.
