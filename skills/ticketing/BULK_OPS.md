# Bulk Operations on Ticket Frontmatter

Operational tooling for editing many ticket files at once. Separated from `SKILL.md` because these are mechanics, not the skill itself.

## Querying tickets

```bash
# All outstanding work
grep -rl '^status: Ready' plans/

# All complete
grep -rl '^status: Complete' plans/

# Pending (awaiting human approval)
grep -rl '^status: Pending' plans/
```

Ready tickets with titles:

```python
python3 -c "
import glob, re
for f in sorted(glob.glob('plans/*/*.md')):
    with open(f) as fh:
        text = fh.read()
    if not text.startswith('---'): continue
    block = text.split('---')[1]
    status = re.search(r'^status:\s*(.+)', block, re.MULTILINE)
    tid = re.search(r'^id:\s*(.+)', block, re.MULTILINE)
    title = re.search(r'^title:\s*\"?(.+?)\"?\s*$', block, re.MULTILINE)
    if status and tid and title and status.group(1).strip() == 'Ready':
        print(f'  {tid.group(1).strip():20s} {title.group(1).strip()}')
"
```

## Bulk frontmatter updates

Do not edit ticket frontmatter one file at a time when updating N tickets to the same state. Use a YAML-aware bulk-edit one-liner. It is faster, atomic per file, and avoids the Read-before-Edit friction of tool-based editing when flipping many tickets at once.

### Status flips (regex is fine)

```bash
# Flip every Pending ticket in an epic to Ready (bulk promotion).
python3 -c "
import re, glob, pathlib
for f in glob.glob('plans/<EPIC>/*.md'):
    p = pathlib.Path(f)
    text = p.read_text()
    new = re.sub(r'^status:\s*Pending\s*$', 'status: Ready', text, count=1, flags=re.MULTILINE)
    if new != text: p.write_text(new)
"

# Flip specific IDs from Ready to Complete.
python3 -c "
import re, pathlib
for tid in ['AUTH-2', 'AUTH-5', 'AUTH-6']:
    p = pathlib.Path(f'plans/AUTH/{tid}.md')
    text = p.read_text()
    new = re.sub(r'^status:\s*Ready\s*$', 'status: Complete', text, count=1, flags=re.MULTILINE)
    if new != text: p.write_text(new)
"

# Verify what changed (run before committing).
git diff plans/
```

### Structured edits (use PyYAML, not regex)

For changes beyond simple status flips — adding `superseded_by`, retargeting `target_repo`, updating `confidence` based on spike outcomes — use ruamel.yaml to preserve formatting and comments:

```python
# Requires: pip install ruamel.yaml
from ruamel.yaml import YAML
import pathlib

yaml = YAML()
for p in pathlib.Path('plans/AUTH').glob('AUTH-*.md'):
    text = p.read_text()
    if not text.startswith('---\n'): continue
    _, front, body = text.split('---\n', 2)
    data = yaml.load(front)
    # Edit here — e.g.:
    if data.get('target_repo') == 'old-name':
        data['target_repo'] = 'new-name'
    # Write back.
    import io; buf = io.StringIO()
    yaml.dump(data, buf)
    p.write_text(f'---\n{buf.getvalue()}---\n{body}')
```

## Rule

Reaching for a bulk update tool is itself a signal that the plan may be mid-rework. If you're flipping a dozen tickets to `Superseded`, capture *why* in one commit message rather than spreading the rationale across many individual edits. Bulk operations should produce one coherent change with one explanation, not N silent edits.
