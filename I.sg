#!/usr/bin/env bash
set -e
ROOT="ai-login-tool"
rm -rf "$ROOT"
mkdir -p "$ROOT/ai_login_tool" "$ROOT/tests" "$ROOT/.github/workflows"

cat > "$ROOT/pyproject.toml" <<'EOF'
[build-system]
requires = ["setuptools>=42","wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "ai-login-tool"
version = "0.1.0"
description = "Combined AI login script generator, HAR-based exporter, and OpenBullet exporter"
readme = "README.md"
requires-python = ">=3.8"
license = {text = "MIT"}
dependencies = ["click>=8.0","requests>=2.25","PyYAML>=5.4","openai>=0.27"]
EOF

cat > "$ROOT/README.md" <<'EOF'
# AI Login Combined Tool

Combined Python tool (CLI + library) that merges features from Ai-login, AUTO-LOGIN-GENERATOR-HAR-Based, and Ai-openbullet-.

Features:
- Parse HAR files to extract login requests
- Generate login scripts using an LLM backend (OpenAI optional)
- Export configurations to OpenBullet format (use responsibly)

Usage examples:

CLI:
  ai-login-tool parse-har --input example.har --out out_dir
  ai-login-tool generate --source out_dir --model openai --out script.py
  ai-login-tool export-openbullet --input script.py --out openbullet.xml

Environment:
- For OpenAI backend set OPENAI_API_KEY in your environment.

License: MIT
EOF

cat > "$ROOT/LICENSE" <<'EOF'
MIT License

Copyright (c) 2026 Shineabit

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
EOF

cat > "$ROOT/.gitignore" <<'EOF'
__pycache__/
*.pyc
.env
.env.*
.vscode/
dist/
build/
*.egg-info
EOF

cat > "$ROOT/ai_login_tool/__init__.py" <<'EOF'
__version__ = "0.1.0"
EOF

cat > "$ROOT/ai_login_tool/cli.py" <<'EOF'
import os
import json
import click
from .har_parser import parse_har
from .ai_generator import generate_script
from .openbullet import export_openbullet

@click.group()
def cli():
    """AI Login Combined Tool CLI"""
    pass

@cli.command()
@click.option('--input', 'input_file', required=True, help='Input HAR file')
@click.option('--out', 'out_dir', required=True, help='Output directory')
def parse_har_cmd(input_file, out_dir):
    os.makedirs(out_dir, exist_ok=True)
    entries = parse_har(input_file)
    out_path = os.path.join(out_dir, 'extracted.json')
    with open(out_path, 'w') as f:
        json.dump(entries, f, indent=2)
    click.echo(f'Parsed HAR and wrote {out_path}')

@cli.command()
@click.option('--source', 'source', required=True, help='Source HAR output dir or file')
@click.option('--model', 'model', default='openai', help='LLM backend: openai or none')
@click.option('--out', 'out_file', required=True, help='Output script file')
def generate_cmd(source, model, out_file):
    # source can be a JSON file produced by parse_har or a HAR
    if source.endswith('.har'):
        entries = parse_har(source)
    else:
        with open(source) as f:
            entries = json.load(f)
    script = generate_script(entries, model=model)
    with open(out_file, 'w') as f:
        f.write(script)
    click.echo(f'Wrote generated script to {out_file}')

@cli.command()
@click.option('--input', 'input_file', required=True, help='Input script or config JSON')
@click.option('--out', 'out_file', required=True, help='Output OpenBullet XML file')
def export_openbullet_cmd(input_file, out_file):
    with open(input_file) as f:
        data = f.read()
    xml = export_openbullet(data)
    with open(out_file, 'w') as f:
        f.write(xml)
    click.echo(f'Wrote OpenBullet config to {out_file}')

if __name__ == '__main__':
    cli()
EOF

cat > "$ROOT/ai_login_tool/har_parser.py" <<'EOF'
import json

def parse_har(har_path):
    """Parse a HAR file and return a list of potential login requests.
    Each entry is a dict with url, method, headers, postData (if any).
    """
    with open(har_path, 'r', encoding='utf8') as f:
        har = json.load(f)
    entries = []
    for e in har.get('log', {}).get('entries', []):
        req = e.get('request', {})
        method = req.get('method')
        url = req.get('url')
        headers = {h['name']: h['value'] for h in req.get('headers', [])}
        postData = req.get('postData', {}).get('text') if req.get('postData') else None
        # Heuristic: POST with username/password-like fields
        candidate = False
        if method == 'POST' and postData:
            lower = postData.lower()
            if 'user' in lower or 'email' in lower or 'pass' in lower:
                candidate = True
        if candidate:
            entries.append({'url': url, 'method': method, 'headers': headers, 'postData': postData})
    return entries
EOF

cat > "$ROOT/ai_login_tool/ai_generator.py" <<'EOF'
import os
import json

# Optional OpenAI integration; if OPENAI_API_KEY is set and model is 'openai', it will call the API.
try:
    import openai
except Exception:
    openai = None

TEMPLATE = '''# Generated login script
import requests

def login(session=None):
    s = session or requests.Session()
{requests_code}
    return s

if __name__ == '__main__':
    s = login()
    print('Done')
'''

def _build_requests_code(entries):
    lines = []
    for i, e in enumerate(entries):
        url = e.get('url')
        post = e.get('postData')
        headers = e.get('headers') or {}
        lines.append(f"    # Request {i}: {url}")
        if post:
            try:
                parsed = json.loads(post)
                lines.append(f"    r = s.post('{url}', json={json.dumps(parsed, indent=8)})")
            except Exception:
                lines.append(f"    r = s.post('{url}', data={json.dumps(post)})")
        else:
            lines.append(f"    r = s.get('{url}')")
        lines.append("    # handle response if needed")
    return '\\n'.join(lines)

def generate_script(entries, model='openai'):
    \"\"\"Generate a Python login script from parsed entries. If model=='openai' and OPENAI_API_KEY is set,
    attempt to ask the LLM to produce an improved script; otherwise use a template builder.
    \"\"\"
    if model == 'openai' and os.environ.get('OPENAI_API_KEY') and openai is not None:
        prompt = 'Generate a Python requests-based login script from these request entries:\\n' + json.dumps(entries)
        openai.api_key = os.environ.get('OPENAI_API_KEY')
        try:
            resp = openai.ChatCompletion.create(
                model='gpt-3.5-turbo',
                messages=[{'role': 'user', 'content': prompt}],
                max_tokens=800
            )
            text = resp['choices'][0]['message']['content']
            return text
        except Exception:
            pass
    requests_code = _build_requests_code(entries)
    return TEMPLATE.format(requests_code=requests_code)
EOF

cat > "$ROOT/ai_login_tool/openbullet.py" <<'EOF'
import xml.etree.ElementTree as ET

def export_openbullet(script_text):
    """Create a very basic OpenBullet-like XML wrapper for the given script or config text.
    This is a minimal serializer and may need adjustments for real OpenBullet versions.
    """
    root = ET.Element('OpenBulletConfigs')
    conf = ET.SubElement(root, 'Config')
    name = ET.SubElement(conf, 'Name')
    name.text = 'Generated by ai-login-tool'
    body = ET.SubElement(conf, 'Script')
    body.text = script_text
    return ET.tostring(root, encoding='unicode')
EOF

cat > "$ROOT/ai_login_tool/utils.py" <<'EOF'
import json

def load_json(path):
    with open(path) as f:
        return json.load(f)

# simple helper
EOF

cat > "$ROOT/tests/test_har_parser.py" <<'EOF'
import json
from ai_login_tool.har_parser import parse_har

def test_parse_har_simple(tmp_path):
    har = {'log': {'entries': [{'request': {'method': 'POST', 'url': 'https://example.com/login', 'headers': [], 'postData': {'text': 'username=admin&password=secret'}}}, {'request': {'method': 'GET', 'url': 'https://example.com/', 'headers': []}}]}}
    p = tmp_path / 't.har'
    p.write_text(json.dumps(har))
    entries = parse_har(str(p))
    assert len(entries) == 1
    assert entries[0]['url'] == 'https://example.com/login'
EOF

cat > "$ROOT/tests/test_ai_generator.py" <<'EOF'
from ai_login_tool.ai_generator import generate_script

def test_generate_script_basic():
    entries = [{'url': 'https://example.com/login', 'method': 'POST', 'postData': 'username=admin&password=secret', 'headers': {}}]
    script = generate_script(entries, model='none')
    assert 'requests' in script
EOF

cat > "$ROOT/tests/test_openbullet.py" <<'EOF'
from ai_login_tool.openbullet import export_openbullet

def test_export_openbullet_simple():
    xml = export_openbullet('print("hello")')
    assert '<OpenBulletConfigs>' in xml
EOF

cat > "$ROOT/.github/workflows/ci.yml" <<'EOF'
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Install
        run: |
          python -m pip install --upgrade pip
          pip install click requests PyYAML openai pytest
      - name: Run tests
        run: pytest -q
EOF

cd "$ROOT"
zip -r ../ai-login-tool.zip .
echo "Done! You now have ai-login-tool.zip to upload to your GitHub repo."
