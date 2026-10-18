# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is the `jbox-web` fork of `saltstack-formulas/salt-formula`. The parent
`../CLAUDE.md` (collection-wide: fork policy, CI, lint, Kitchen commands, the
template layout) applies here; this file only covers what is specific to this
formula.

## Commands

```bash
bundle install
bin/kitchen list                                # 3 instances: suite `default` x 3 platforms
bin/kitchen test default-debian-13-3008-2       # also default-debian-13-3006-27, default-debian-11-3006-27
bin/kitchen converge default-debian-11-3006-27  # keep the container to inspect /etc/salt
pre-commit run --all-files
```

`Vagrantfile`, `kitchen.vagrant.yml`, `kitchen.macos.yml`, `kitchen.windows.yml`
and `dev/` are upstream leftovers: no CI runs them.

## What the fleet uses

Only `salt.master` (the salt host) and `salt.minion` (every host), driven by
`salt:master:*` and `salt:minion:*` pillar keys. `salt.pkgrepo`, `salt.formulas`,
`salt.gitfs.*`, `salt.cloud`, `salt.ssh`, `salt.api`, `salt.syndic`,
`salt.standalone` and every non-Debian branch reach no host.

## Architecture — how it departs from the template

This formula predates the template layout described in `../CLAUDE.md`:

- **Flat states.** No `package/` / `config/` / `service/` split. `master.sls`,
  `minion.sls`, `standalone.sls`… each carry `pkg.installed` + `file.recurse` +
  `service.*` under one ID (`salt-master`, `salt-minion`). `init.sls` always
  includes `salt.pkgrepo`, and each other state only if its pillar section is
  present (`salt:minion:master_type` picks `minion` over `standalone`).
- **`map.jinja` exports two dicts**: `salt_settings` (`defaults.yaml:salt` <-
  `osfamilymap` <- `osmap` <- `osfingermap` <- `salt:lookup` <- `salt`) and
  `formulas_settings` (pillar `salt_formulas`). No `osarchmap`; merging goes
  through `defaults.merge`. `osmap.yaml` and `osfingermap.yaml` have no Debian
  entry.
- **Ordering by `order:`**, not requisites, for the repo and the pin:
  `pin.sls` (apt preference, Debian only, when `salt:version` and
  `salt:pin_version` are set) is `order: 2`, `pkgrepo/debian/install.sls` is
  `order: 3`, because the formula cannot know which of `salt-master` /
  `salt-minion` exists to `require_in` it. Services are `order: last`.

### Config rendering — the part that matters

The real templates are `salt/files/master.d/*.conf` and
`salt/files/minion.d/*.conf`, rendered with `template: jinja` by a
`file.recurse` into `/etc/salt/{master,minion}.d`.

- `f_defaults.conf` (1833 lines master, 1270 minion) walks the upstream
  2016.11 default config. Its `get_config(name, default)` macro emits, in order
  of precedence: `salt:master:<name>` (or `salt:minion:<name>`), then the
  top-level `salt:<name>` unless it is in `reserved_keys`, else the default as a
  comment. Only pillar-set keys become live config.
- Keys absent from the template are appended by a passthrough loop at the end
  of the file. The master loop uses `yaml(False)`; the minion loop writes list
  items raw (`- {{ item }}`): an item containing `: ` or `#`, or starting with
  `*`, `&` or `!`, renders invalid or altered YAML.
- `salt/files/default/{master,minion}.d/` are **not** the templates: they are
  4-line TOFS placeholders, used only when `master_config_use_TOFS` /
  `minion_config_use_TOFS` is true (default false). Edit `salt/files/master.d/`
  and `salt/files/minion.d/`.
- `clean: {{ clean_config_d_dir }}` (default true) deletes every file in the
  `*.d` directory the formula does not ship, except `_*` (and `raas.conf` on the
  master). Adding a template file creates it on every host running the state;
  renaming one deletes the old name.
- Any change to `minion.d` restarts the minion **during** the highstate through
  a backgrounded `salt-call --local service.restart` (`restart_via_at: false`).
- `formulas.jinja` provides `file_roots()`, called from both `f_defaults.conf`
  to append `salt_formulas:list` checkouts to `file_roots`; `formulas.sls` clones
  them with `git.latest`.

## Tests

- One suite, `default`, whose `state_top` is `salt._mapdata`, `salt.master`,
  `salt.minion`. The Salt apt repository comes from the image, as it comes from
  apt-formula in production.
- `test/salt/pillar/salt.sls` pins `salt:version` to the image's
  `saltversion`, so the converge does not upgrade the running onedir in place
  (that breaks the post-install module refresh). The `_mapdata` control drops
  `salt_settings.version` on both sides for the same reason.
- Of the 44 reference dumps in `test/integration/default/files/_mapdata/`,
  only `debian-11.yaml` and `debian-13.yaml` are exercised; a change to
  `defaults.yaml`, `osfamilymap.yaml` (Debian) or the test pillar must be carried
  into both. `top.sls` still targets `v3003-py3` / `v3004-py3` pillars that no
  platform matches.
- InSpec controls only check that `salt-master` / `salt-minion` are installed
  and running, plus the `_mapdata` comparison. Nothing asserts the rendered
  `master.d` / `minion.d` content: verify a template change by converging and
  reading `/etc/salt/*.d/f_defaults.conf` in the container.
