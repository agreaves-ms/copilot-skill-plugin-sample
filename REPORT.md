# One marketplace, one source per topic

**Use a real plugin root per topic and a real aggregate manifest above them.**
On GitHub Copilot CLI **1.0.114-0** (macOS arm64, November 18, 2026), a single
marketplace installed either topic independently or both through the aggregate.
The aggregate loaded the **same checked-in skills and agents**, including a
skill's relative reference. No symlinks, generated copies, dependency resolver,
or installer modifications were needed.

**Offer the aggregate *or* selected topics.** Installing both is allowed, but
does not collapse their shared content into one installed plugin or one runtime
name. In the overlap, a bare skill invocation actually failed as ambiguous.
Also, shipping an instruction file is not proof the CLI applies it. These are
the two qualifications to a migration, not reasons to duplicate source files.

## What to change in HVE

Keep the existing all-in-one plugin **name** and use one marketplace entry for
it, plus one per topic. Put each canonical skill (with its references) and
agent within the topic plugin that owns it. Use relative paths *down* from
the aggregate manifest to list the same content. A topic installer then needs
only its subtree. If moving existing paths would break links, inventory those
links first: this test is not evidence that an arbitrary existing flat tree can
be split merely by changing marketplace entries.

The [catalog](.github/plugin/marketplace.json), [aggregate](plugin.json) and
[Amber manifest](topics/amber/plugin.json) are a working example. The other
catalog entries are **experiments, not examples to ship**. For a large repo,
consider a common packaging directory instead of making the entire repo the
aggregate payload. Keep unrelated source and build artifacts outside that
directory. Maintain a small manifest-membership check in CI: every intended
topic skill/agent should resolve from its topic and from the aggregate; also
smoke-test both independently from outside the checkout. This duplicates
path declarations, not the content. Use explicit skill directories and agent
files where possible; see the VS Code qualification below.

This solves overlap **between a topic and its parent aggregate**. It does not
give two disjoint topic roots access to the same external helper or instruction.
Keep a skill's required references inside its own topic and check cross-topic
links and agent delegations before declaring any focused install self-contained.

## CLI results: remote marketplace, fresh processes

The [baseline revision](https://github.com/agreaves-ms/copilot-skill-plugin-sample/tree/9070b82e8839384b34d52312ef725e9fcad1f085)
and [additional controls](https://github.com/agreaves-ms/copilot-skill-plugin-sample/tree/51aa730e1a0ab847bcbc5a6b883c716948a7f8a7)
were pushed **before** installation. The CLI fetched the sample repository from
GitHub by SHA. Nothing below relies on running Copilot in the sample checkout.

| What the marketplace entry selected | Installed and discoverable result |
|---|---|
| Independent roots `./topics/amber`, `./topics/indigo`, each with `plugin.json` | Each listed only its own skill. Indigo's catalog exposed only its own agent; Amber's namespaced agent returned its own sentinel. |
| `prism-deps`, no components, `dependencies: ["prism-amber","prism-indigo"]` in both plugin and entry | Install succeeded; only `prism-deps` was installed, with zero plugin skills and no available sample agents. Neither sibling was pulled in. |
| Two entries sharing manifestless `./topics`, per-entry `skills` and `agents`, `strict: false` | Each reported success and “Installed 1 skill.” Both topic subtrees were copied to each install. Neither entry exposed a skill; Amber's agent catalog was empty. Installing both did not fix discovery. |
| Same shared source with `strict: true`; separate `false` control specifying an individual skill directory and agent file | Again, no plugin skills in either case; the individual-path control had no agents. |
| Root source with real `prism-all` manifest, but Amber-only component selections in the marketplace entry (`prism-overlay`, `prism-overlay-strict`) | Separate install records under the entry names. Both loaded **both** skills, not just Amber. The strict overlay exposed both agents under its own entry name. Neither value of `strict` acted as a topic filter. |
| Root `./` with real aggregate manifest selecting nested topics | Two skills and two invokable agents. Amber used a skill collection and agent directory; Indigo used a skill leaf directory and an individual `.agent.md` file. No topic-plugin installation record was created. |

As a separate local control, `copilot plugin install "$CHECKOUT/topics/amber"`
and, after uninstalling it, `copilot plugin install "$CHECKOUT/topics/indigo"`
each yielded one matching skill and agent in the neutral process. These were
direct installs without a marketplace identity. Both commands warned that
direct installs are deprecated; use the marketplace for distribution.

### Co-installation, reinstall, and ownership

| Sequence (each row starts in its indicated state) | Native skill inventory, excluding built-ins |
|---|---|
| Clean → Amber → All | `A` → `A, A, I` |
| Both → install Amber again | Still two plugin records and `A, A, I`; order changed, not cardinality. |
| Both → uninstall Amber → reinstall Amber → uninstall All → uninstall Amber | `A, I` → `A, A, I` → `A` → empty |
| Another clean profile → All → Amber → uninstall All | `A, I` → `A, A, I` → `A` |
| Remaining Amber → reinstall All → uninstall Amber → uninstall All → install Indigo | `A, A, I` → `A, I` → empty → `I` |

`A` and `I` denote the bare skill names `prism-amber-card` and
`prism-indigo-card`. The two Amber rows in `skill list --json` had **different
installed paths**. There is one Amber `SKILL.md` in Git. All-only installed a
snapshot including the catalog, topic manifests, controls, and both topics;
Amber-only installed its subtree. With both, the overlapping file was present
under both `installed-plugins/prism-lab/prism-amber/skills/…` and
`installed-plugins/prism-lab/prism-all/topics/amber/skills/…`. Uninstalling
either did not remove the survivor's file or its discovery record.

**Actual invocations**, not installer counts:

- The skill tool returned `Skill "prism-amber-card" is ambiguous. Use one of:
  prism-all:prism-amber-card, prism-amber:prism-amber-card`. Both explicitly
  qualified tool calls succeeded in that same session. With either install
  order the model could choose the focused qualified name; do not rely on
  install order to resolve an unqualified reference.
- A `skill` invocation for Indigo followed by `view` of the installed
  `topics/indigo/skills/prism-indigo-card/references/card.md` returned
  `INDIGO-2853`. The Amber reference returned `AMBER-4716`. These are fixture
  values stored in the cards, not supplied in the requests.
- `--agent prism-all:prism-amber-reader` and `--agent prism-amber:prism-amber-reader`
  each returned `READER-A914`; `--agent prism-all:prism-indigo-reader` returned
  `READER-I683`. A nonexistent `prism-amber:prism-indigo-reader` exited 1
  and listed exactly those three available IDs in the combined profile.

### Instructions are a separate compatibility gate

The Amber root contained `rules/prism-amber.instructions.md`, with `applyTo:
"**"`. Topic and aggregate manifests both tried `instructions` and `rules`
paths. `copilot instruction list --json` returned `[]`; a model call with no
file tools and no identifier in the question answered `unknown`, not the rule
sentinel. A separate [Agent Plugins 1.0 control](controls/standard) placed a
different instruction under the documented `com.github.copilot/rules/` path.
That file was installed; the same inventory and no-tools check also yielded
`[]` and `unknown`. In a separate positive control, the inventory did list
a temporary `$COPILOT_HOME/copilot-instructions.md` as `home-copilot`. The
model's reply alone is not a reliable test of whether instructions loaded.
This is a negative observation for these files on this
build, **not** a claim that no client supports plugin rules. A reference read
on skill activation is useful for opt-in guidance but is not an always-applied
instruction. Do not promise parity for required HVE instructions without
testing their actual application conditions on each supported client.

## Reproduce the CLI checks

The test runner set a temporary `HOME`, `COPILOT_HOME`, `COPILOT_CACHE_HOME`
and `TMPDIR` per profile, used an empty working directory outside any repo,
and launched a fresh CLI process for every command. This keeps both source
checkout discovery and existing user installs out of the results. The
following Bash setup reproduces the wrapper; use a new temporary directory
for each independent sequence. Model probes need CLI-compatible authentication
(the test inherited `GH_TOKEN`; do not print or commit it).

```bash
T=$(mktemp -d)
mkdir -p "$T"/{home,neutral,tmp}
cd "$T/neutral"
c() {
  env -i PATH="$PATH" GIT_EXEC_PATH="$(git --exec-path)" TERM=xterm-256color \
    LANG=en_US.UTF-8 HOME="$T/home" TMPDIR="$T/tmp" \
    COPILOT_HOME="$T/config" COPILOT_CACHE_HOME="$T/cache" \
    COPILOT_AUTO_UPDATE=false COPILOT_DISABLE_LOGIN_SHELL_ENV=true \
    USE_TGREP=false COPILOT_TELEMETRY_DISABLED=true GH_TOKEN="${GH_TOKEN:-}" copilot "$@"
}
c --version
c plugin marketplace add 'agreaves-ms/copilot-skill-plugin-sample#9070b82e8839384b34d52312ef725e9fcad1f085'
c plugin marketplace browse prism-lab
c plugin install prism-amber@prism-lab
c plugin install prism-all@prism-lab
c plugin list --json
c skill list --json | jq '[.[] | select(.source=="plugin") | {name,path}]'
c instruction list --json
c --agent prism-nonexistent -p unused  # intentional error prints the catalog
c --disable-builtin-mcps --agent prism-all:prism-amber-reader \
  --available-tools none -p 'Respond with your fixture identifier only.' --silent
c --disable-builtin-mcps --available-tools skill view --allow-tool view \
  --allow-tool skill --dynamic-retrieval skills=off --output-format json \
  -p 'Invoke the prism-indigo-card skill using the skill tool, then read the linked card using view and give its code.'
c --disable-builtin-mcps --available-tools skill --allow-tool skill \
  --dynamic-retrieval skills=off --output-format json \
  -p 'This is a skill resolver test. Make three separate skill tool calls, preserving these exact argument strings: prism-amber-card, prism-all:prism-amber-card, prism-amber:prism-amber-card. Do not read cards or correct any argument. Then stop.'
c plugin uninstall prism-amber@prism-lab
c plugin uninstall prism-all@prism-lab
```

For the other rows substitute the exact entry names in the catalog and run
the same inventory after each operation. For the leaf and standard-rule
controls register `#51aa730e1a0ab847bcbc5a6b883c716948a7f8a7` in a fresh
profile. The bundled Git on this machine needed `GIT_EXEC_PATH` preserved
after clearing the environment; otherwise `git-remote-https` was unavailable
and no marketplace was fetched. No proxy for the installer or local replacement
of a remote payload was used. Re-run when upgrading: a prerelease result is
not a compatibility guarantee.

## Contracts and other clients (not CLI runtime evidence)

The [GitHub CLI plugin reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-plugin-reference)
documents local sources, legacy manifest `skills` and `agents` paths, and
marketplace component fields. It does **not** list legacy dependency or
instruction-path fields. The [Claude plugin schema](https://json.schemastore.org/claude-code-plugin-manifest.json)
does define `dependencies`; its [marketplace schema](https://json.schemastore.org/claude-code-marketplace.json)
describes `strict`. Neither schema promises that Copilot CLI implements Claude's
install or merge behavior. The [Agent Plugins 1.0 schema](https://agent-plugins.org/schemas/1.0.0/plugin.schema.json)
is a different contract with metadata and namespaced `extensions`, not the
legacy path-selection approach validated here. The CLI reference documents
the namespaced rules directory for this format. Sources consulted 2026-11-18.

**VS Code — source inspection, not an editor installation test.** At
[`microsoft/vscode@1cbf1bb`](https://github.com/microsoft/vscode/tree/1cbf1bb378524fefc7e5c9a52a75cf67ba1d2509),
the [marketplace source resolver](https://github.com/microsoft/vscode/blob/1cbf1bb378524fefc7e5c9a52a75cf67ba1d2509/src/vs/workbench/contrib/chat/common/plugins/pluginMarketplaceService.ts#L104-L155)
distinguishes the marketplace identity from the repository/ref/path used to
locate a plugin. The [component loader](https://github.com/microsoft/vscode/blob/1cbf1bb378524fefc7e5c9a52a75cf67ba1d2509/src/vs/workbench/contrib/chat/common/plugins/agentPluginServiceImpl.ts#L400-L469)
reads a manifest at the plugin URI and observes the `skills`, `agents` and
`rules` components; it does not treat the mere presence of arbitrary Markdown
in the downloaded tree as registration. This supports testing distinct plugin
roots and their own manifests in VS Code. It is not proof of editor install,
rule application, or overlap behavior; those CLI results must not be presented
as VS Code results.
