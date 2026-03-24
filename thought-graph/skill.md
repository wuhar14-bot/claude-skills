---
name: thought-graph
description: Set up and launch a local D3.js graph viewer for thought-recording + focus-session files. Creates server.py + index.html if not present, then starts the server. Use when user says "thought graph", "open graph", "graph viewer", "launch graph".
type: setup+launch
---

# Skill: Thought Graph Viewer

**Triggers**: `thought graph`, `open graph`, `graph viewer`, `launch graph`, `start graph`

A local Obsidian-style knowledge graph that visualises your `thoughts_YYYY-MM-DD.md` daily notes and `session N.md` focus-session files as an interactive D3.js force graph.

---

## Step 1 — Detect configuration

Ask the user (or infer from context) for:
- `BASE_DIR`: root of the Obsidian vault / note folder (e.g. `E:/claude-code/2025_new_me_copy`)
- `GRAPH_DIR`: where to put the graph viewer files (default: `BASE_DIR/graph-viewer`)
- `PORT`: default `3001`

Expected folder structure inside BASE_DIR:
```
BASE_DIR/
  thought-recording/     ← thoughts_YYYY-MM-DD.md files
  focus-sessions/        ← YYYY-MM/YYYY-MM-DD/session N.md files
  graph-viewer/          ← created by this skill
    server.py
    index.html
```

If the user already has `graph-viewer/server.py`, skip to Step 3 (just start).

---

## Step 2 — Create files

### 2A — Create `GRAPH_DIR/server.py`

Write the following content, substituting `BASE_DIR`, `THOUGHT_DIR`, `SESSION_DIR`, `PORT`:

```python
#!/usr/bin/env python3
"""
Thought Graph Viewer — server.py
Run: python server.py
Then open: http://localhost:PORT
"""

import json
import os
import re
import webbrowser
import threading
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs, unquote
from pathlib import Path

BASE_DIR    = Path("BASE_DIR")
THOUGHT_DIR = BASE_DIR / "thought-recording"
SESSION_DIR = BASE_DIR / "focus-sessions"
PORT        = PORT_NUMBER
INDEX_HTML  = Path(__file__).parent / "index.html"


def norm(path_str):
    return str(path_str).replace("\\", "/")


def parse_thoughts_file(path):
    path = Path(path)
    date = path.stem.replace("thoughts_", "")
    node_id = f"thought:{date}"
    nodes = [{"id": node_id, "label": date, "type": "thought", "path": norm(path)}]
    edges = []
    try:
        content = path.read_text(encoding="utf-8")
    except Exception:
        return nodes, edges

    block_pattern = re.compile(
        r'\*\*Session (\d+)\*\*[^\n]+\n'
        r'(?:📋 Session Note: `([^`]+)`\n)?'
        r'(?:🎯 Main Output: `([^`]+)`)?',
        re.MULTILINE
    )
    for m in block_pattern.finditer(content):
        session_path = m.group(2)
        ko_path      = m.group(3)
        session_title = m.group(0).splitlines()[0]

        if session_path:
            session_path = session_path.strip()
            session_id   = f"session:{norm(session_path)}"
            edges.append({"source": node_id, "target": session_id})
            if ko_path:
                ko_path = ko_path.strip()
                ko_id   = f"key_output:{norm(ko_path)}"
                edges.append({"source": session_id, "target": ko_id,
                               "_ko_desc": session_title})
    return nodes, edges


def parse_session_file(path):
    path = Path(path)
    num_match = re.search(r'session\s+(\d+)', path.name, re.IGNORECASE)
    num       = num_match.group(1) if num_match else "?"
    date_part = path.parent.name
    session_id = f"session:{norm(path)}"
    nodes, edges = [], []
    try:
        content = path.read_text(encoding="utf-8")
    except Exception:
        return nodes, edges

    title_match = re.search(r'^# (.+)$', content, re.MULTILINE)
    title = title_match.group(1).strip() if title_match else f"Session {num}"

    project = None
    yaml_match = re.match(r'^---\s*\n(.*?)\n---', content, re.DOTALL)
    if yaml_match:
        pm = re.search(r'^project:\s*(.+)$', yaml_match.group(1), re.MULTILINE)
        if pm:
            project = pm.group(1).strip()
    if not project:
        first_lines = "\n".join(content.splitlines()[:15])
        pm = re.search(r'^project:\s*(.+)$', first_lines, re.MULTILINE)
        if pm:
            project = pm.group(1).strip()

    node = {"id": session_id, "label": f"S{num} · {date_part}",
            "type": "session", "path": norm(path), "title": title}
    if project:
        node["project"] = project
        edges.append({"source": session_id, "target": f"project:{project}"})
    nodes.append(node)
    return nodes, edges


def build_graph():
    all_nodes, all_edges = {}, []

    for tf in sorted(THOUGHT_DIR.glob("thoughts_*.md")):
        nodes, edges = parse_thoughts_file(tf)
        for n in nodes:
            all_nodes[n["id"]] = n
        all_edges.extend(edges)

    for sf in sorted(SESSION_DIR.rglob("session *.md")):
        nodes, edges = parse_session_file(sf)
        for n in nodes:
            all_nodes[n["id"]] = n
        all_edges.extend(edges)

    for edge in list(all_edges):
        sid = edge["target"]
        if sid.startswith("session:") and sid not in all_nodes:
            raw_path = sid[len("session:"):]
            fname = raw_path.split("/")[-1]
            all_nodes[sid] = {"id": sid, "label": fname, "type": "session",
                               "path": raw_path, "title": fname}

    for sid, snode in list(all_nodes.items()):
        if snode["type"] != "session":
            continue
        raw_path = snode.get("path", "")
        parts = raw_path.replace("\\", "/").split("/")
        date_part = None
        for part in reversed(parts[:-1]):
            if re.match(r'^\d{4}-\d{2}-\d{2}$', part):
                date_part = part
                break
        if not date_part:
            fname = parts[-1]
            m = re.search(r'(\d{4})(\d{2})(\d{2})', fname)
            if m:
                date_part = f"{m.group(1)}-{m.group(2)}-{m.group(3)}"
        if date_part:
            thought_id = f"thought:{date_part}"
            if thought_id in all_nodes:
                all_edges.append({"source": thought_id, "target": sid})

    for n in all_nodes.values():
        if n.get("project"):
            pid = f"project:{n['project']}"
            if pid not in all_nodes:
                all_nodes[pid] = {"id": pid, "label": n["project"], "type": "project"}

    ko_descs = {}
    for edge in all_edges:
        if edge.get("_ko_desc") and edge["target"].startswith("key_output:"):
            ko_descs[edge["target"]] = edge["_ko_desc"]

    for edge in all_edges:
        kid = edge["target"]
        if kid.startswith("key_output:") and kid not in all_nodes:
            file_path = kid[len("key_output:"):]
            fname = file_path.split("/")[-1]
            node = {"id": kid, "label": fname, "type": "key_output", "path": file_path}
            if kid in ko_descs:
                node["description"] = ko_descs[kid]
            all_nodes[kid] = node

    seen, unique_edges = set(), []
    for e in all_edges:
        key = (e["source"], e["target"])
        if key not in seen and e["source"] in all_nodes and e["target"] in all_nodes:
            seen.add(key)
            unique_edges.append(e)

    return {"nodes": list(all_nodes.values()), "edges": unique_edges}


_graph_cache = None
_cache_mtime = 0


def get_graph():
    global _graph_cache, _cache_mtime
    latest = 0
    for d in [THOUGHT_DIR, SESSION_DIR]:
        if d.exists():
            for f in d.rglob("*.md"):
                try:
                    latest = max(latest, f.stat().st_mtime)
                except Exception:
                    pass
    if _graph_cache is None or latest > _cache_mtime:
        print("[server] Rebuilding graph...")
        _graph_cache = build_graph()
        _cache_mtime = latest
        n = len(_graph_cache["nodes"])
        e = len(_graph_cache["edges"])
        print(f"[server] Graph ready: {n} nodes, {e} edges")
    return _graph_cache


class Handler(BaseHTTPRequestHandler):
    def log_message(self, format, *args):
        pass

    def send_json(self, data, status=200):
        body = json.dumps(data, ensure_ascii=False).encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", len(body))
        self.send_header("Access-Control-Allow-Origin", "*")
        self.end_headers()
        self.wfile.write(body)

    def send_text(self, text, status=200):
        body = text.encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "text/plain; charset=utf-8")
        self.send_header("Content-Length", len(body))
        self.send_header("Access-Control-Allow-Origin", "*")
        self.end_headers()
        self.wfile.write(body)

    def send_html(self, html, status=200):
        body = html if isinstance(html, bytes) else html.encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.send_header("Content-Length", len(body))
        self.end_headers()
        self.wfile.write(body)

    def do_GET(self):
        parsed = urlparse(self.path)
        path = parsed.path
        qs = parse_qs(parsed.query)

        if path in ("/", "/index.html"):
            try:
                self.send_html(INDEX_HTML.read_bytes())
            except FileNotFoundError:
                self.send_html(b"<h1>index.html not found</h1>", 404)

        elif path == "/api/graph":
            try:
                self.send_json(get_graph())
            except Exception as ex:
                self.send_json({"error": str(ex)}, 500)

        elif path == "/api/file":
            file_path = unquote(qs.get("path", [""])[0])
            raw_mode  = qs.get("raw", [""])[0] == "1"
            try:
                fp  = Path(file_path)
                ext = fp.suffix.lower()
                binary = {".docx",".pptx",".xlsx",".pdf",".doc",".xls",
                          ".png",".jpg",".jpeg",".gif",".zip"}
                if ext in binary:
                    self.send_text(f"Binary file: {file_path}", 415)
                    return
                content = fp.read_text(encoding="utf-8")
                if raw_mode and ext in {".html", ".htm"}:
                    self.send_html(content)
                else:
                    self.send_text(content)
            except FileNotFoundError:
                self.send_text(f"File not found: {file_path}", 404)
            except Exception as ex:
                self.send_text(str(ex), 500)
        else:
            self.send_text("Not found", 404)


def open_browser():
    import time; time.sleep(0.8)
    webbrowser.open(f"http://localhost:{PORT}")


if __name__ == "__main__":
    server = HTTPServer(("localhost", PORT), Handler)
    print(f"[server] Thought Graph running at http://localhost:{PORT}")
    print(f"[server] Base: {BASE_DIR}")
    print("[server] Press Ctrl+C to stop")
    get_graph()
    threading.Thread(target=open_browser, daemon=True).start()
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        print("\n[server] Stopped.")
```

**IMPORTANT when writing the file**: Replace these placeholders with the user's actual values:
- `BASE_DIR` → e.g. `E:/MyVault` or `/Users/alice/notes`
- `PORT_NUMBER` → `3001` (or whatever port the user wants)

---

### 2B — Create `GRAPH_DIR/index.html`

Write the following file verbatim (no substitutions needed):

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Thought Graph</title>
<script src="https://d3js.org/d3.v7.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
<style>
  :root {
    --base:    #1e1e2e; --mantle:  #181825; --crust:   #11111b;
    --surface0:#313244; --surface1:#45475a; --surface2:#585b70;
    --overlay0:#6c7086; --text:    #cdd6f4; --subtext: #a6adc8;
    --blue:    #89b4fa; --green:   #a6e3a1; --peach:   #fab387;
    --lavender:#b4befe; --mauve:   #cba6f7; --red:     #f38ba8;
  }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { background: var(--base); color: var(--text); font-family: 'Segoe UI', system-ui, sans-serif; display: flex; height: 100vh; overflow: hidden; }
  #sidebar { width: 200px; min-width: 200px; background: var(--mantle); border-right: 1px solid var(--surface0); display: flex; flex-direction: column; padding: 16px 12px; gap: 12px; z-index: 10; }
  #sidebar h2 { font-size: 13px; font-weight: 600; color: var(--subtext); letter-spacing: 0.08em; text-transform: uppercase; }
  .filter-group { display: flex; flex-direction: column; gap: 6px; }
  .filter-item { display: flex; align-items: center; gap: 8px; font-size: 13px; cursor: pointer; padding: 4px 6px; border-radius: 6px; transition: background 0.15s; }
  .filter-item:hover { background: var(--surface0); }
  .dot { width: 10px; height: 10px; border-radius: 50%; flex-shrink: 0; }
  .filter-item input[type=checkbox] { accent-color: var(--blue); width: 14px; height: 14px; cursor: pointer; }
  #sidebar hr { border: none; border-top: 1px solid var(--surface0); }
  #stats { font-size: 11px; color: var(--overlay0); line-height: 1.6; }
  #search-box { background: var(--surface0); border: 1px solid var(--surface1); border-radius: 6px; color: var(--text); font-size: 13px; padding: 6px 10px; outline: none; width: 100%; }
  #search-box:focus { border-color: var(--blue); }
  #search-box::placeholder { color: var(--overlay0); }
  #graph-container { flex: 1; position: relative; overflow: hidden; }
  #graph-svg { width: 100%; height: 100%; background: var(--base); }
  #preview-panel { position: fixed; top: 0; right: 0; width: 380px; height: 100vh; background: var(--mantle); border-left: 1px solid var(--surface0); display: flex; flex-direction: column; overflow: hidden; transform: translateX(100%); transition: transform 0.25s ease; z-index: 20; box-shadow: -4px 0 24px rgba(0,0,0,0.4); }
  #preview-panel.open { transform: translateX(0); }
  #preview-header { padding: 12px 16px; background: var(--crust); border-bottom: 1px solid var(--surface0); display: flex; align-items: center; justify-content: space-between; gap: 8px; }
  #preview-title { font-size: 13px; font-weight: 600; color: var(--text); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  #preview-close { background: none; border: none; color: var(--overlay0); font-size: 18px; cursor: pointer; padding: 2px 6px; border-radius: 4px; line-height: 1; }
  #preview-close:hover { background: var(--surface0); color: var(--text); }
  #preview-badge { font-size: 10px; padding: 2px 7px; border-radius: 99px; font-weight: 600; flex-shrink: 0; }
  #preview-body { flex: 1; overflow-y: auto; padding: 16px; font-size: 13px; line-height: 1.7; color: var(--text); }
  #preview-body::-webkit-scrollbar { width: 4px; }
  #preview-body::-webkit-scrollbar-track { background: transparent; }
  #preview-body::-webkit-scrollbar-thumb { background: var(--surface1); border-radius: 2px; }
  #preview-body h1 { font-size: 18px; margin-bottom: 12px; color: var(--lavender); }
  #preview-body h2 { font-size: 15px; margin: 14px 0 6px; color: var(--blue); border-bottom: 1px solid var(--surface0); padding-bottom: 4px; }
  #preview-body h3 { font-size: 13px; margin: 10px 0 4px; color: var(--mauve); }
  #preview-body p  { margin-bottom: 8px; }
  #preview-body ul, #preview-body ol { padding-left: 20px; margin-bottom: 8px; }
  #preview-body li { margin-bottom: 2px; }
  #preview-body code { background: var(--surface0); padding: 1px 5px; border-radius: 4px; font-family: 'Cascadia Code','Fira Code',monospace; font-size: 11px; color: var(--peach); }
  #preview-body pre { background: var(--crust); border: 1px solid var(--surface0); border-radius: 6px; padding: 10px; overflow-x: auto; margin-bottom: 10px; }
  #preview-body pre code { background: none; padding: 0; color: var(--text); }
  #preview-body table { border-collapse: collapse; width: 100%; margin-bottom: 10px; font-size: 12px; }
  #preview-body th, #preview-body td { border: 1px solid var(--surface1); padding: 4px 8px; text-align: left; }
  #preview-body th { background: var(--surface0); color: var(--blue); }
  #preview-body blockquote { border-left: 3px solid var(--mauve); padding-left: 10px; color: var(--subtext); margin-bottom: 8px; }
  #preview-body a { color: var(--blue); text-decoration: none; }
  #preview-body strong { color: var(--peach); }
  #preview-body hr { border: none; border-top: 1px solid var(--surface0); margin: 10px 0; }
  #tooltip { position: fixed; background: var(--surface0); border: 1px solid var(--surface1); border-radius: 8px; padding: 8px 12px; font-size: 12px; color: var(--text); pointer-events: none; display: none; z-index: 100; max-width: 280px; }
  #tooltip .tt-title { font-weight: 600; margin-bottom: 2px; }
  #tooltip .tt-sub   { color: var(--subtext); font-size: 11px; }
  #loading { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; background: var(--base); z-index: 50; flex-direction: column; gap: 12px; font-size: 14px; color: var(--subtext); }
  .spinner { width: 28px; height: 28px; border: 3px solid var(--surface1); border-top-color: var(--blue); border-radius: 50%; animation: spin 0.8s linear infinite; }
  @keyframes spin { to { transform: rotate(360deg); } }
  .link { stroke: var(--surface2); stroke-opacity: 0.5; stroke-width: 1; }
  .link.highlighted { stroke: var(--blue); stroke-opacity: 0.9; stroke-width: 1.5; }
  .node circle { stroke-width: 1.5; cursor: pointer; transition: r 0.15s; }
  .node circle:hover { stroke-width: 2.5; }
  .node.dimmed circle { opacity: 0.15; }
  .node.dimmed text  { opacity: 0.1; }
  .link.dimmed { stroke-opacity: 0.05; }
  .node text { font-size: 10px; fill: var(--subtext); pointer-events: none; user-select: none; }
  .type-project   { fill: var(--peach);   stroke: #c97a4a; }
  .type-thought   { fill: var(--blue);    stroke: #5a8de0; }
  .type-session   { fill: var(--surface2); stroke: var(--overlay0); opacity: 0.4; }
  .type-key_output { fill: #f59e0b; stroke: #d97706; }
</style>
</head>
<body>
<div id="sidebar">
  <h2>Thought Graph</h2>
  <input id="search-box" type="text" placeholder="Search nodes…" autocomplete="off">
  <hr>
  <div class="filter-group">
    <label class="filter-item"><input type="checkbox" id="f-project" checked><span class="dot" style="background:var(--peach)"></span><span>Projects</span></label>
    <label class="filter-item"><input type="checkbox" id="f-thought" checked><span class="dot" style="background:var(--blue)"></span><span>Daily thoughts</span></label>
    <label class="filter-item"><input type="checkbox" id="f-session" checked><span class="dot" style="background:var(--surface2)"></span><span>Sessions</span></label>
    <label class="filter-item"><input type="checkbox" id="f-key_output" checked><span class="dot" style="background:#f59e0b"></span><span>Key outputs</span></label>
  </div>
  <hr>
  <div id="stats">Loading…</div>
</div>
<div id="graph-container">
  <div id="loading"><div class="spinner"></div><span>Building graph…</span></div>
  <svg id="graph-svg"></svg>
</div>
<div id="preview-panel">
  <div id="preview-header">
    <span id="preview-badge">session</span>
    <span id="preview-title">…</span>
    <button id="preview-close">×</button>
  </div>
  <div id="preview-body"></div>
</div>
<div id="tooltip">
  <div class="tt-title" id="tt-title"></div>
  <div class="tt-sub"   id="tt-sub"></div>
</div>
<script>
const NODE_RADIUS = { thought: 11, session: 7, key_output: 5 };
const PROJECT_RADIUS_MIN = 9, PROJECT_RADIUS_MAX = 24;
const TYPE_COLORS  = { project:'var(--peach)', thought:'var(--blue)', session:'var(--surface2)', key_output:'#f59e0b' };
const BADGE_COLORS = { project:'#fab387', thought:'#89b4fa', session:'#585b70', key_output:'#f59e0b' };
let allNodes=[], allEdges=[], visibleTypes=new Set(['project','thought','session','key_output']);
let selectedNode=null, simulation, svg, linkSel, nodeSel, zoom, searchQuery='', draggedNode=null;

async function loadGraph() {
  const res = await fetch('/api/graph');
  const data = await res.json();
  allNodes = data.nodes; allEdges = data.edges;
  document.getElementById('loading').style.display = 'none';
  buildGraph(); updateStats();
}
function getVisibleNodes() {
  let nodes = allNodes.filter(n => visibleTypes.has(n.type));
  if (searchQuery) {
    const q = searchQuery.toLowerCase();
    nodes = nodes.filter(n => n.label.toLowerCase().includes(q) || (n.title||'').toLowerCase().includes(q) || (n.project||'').toLowerCase().includes(q));
  }
  return nodes;
}
function getVisibleEdges(nodeSet) {
  const ids = new Set(nodeSet.map(n => n.id));
  return allEdges.filter(e => ids.has(e.source.id??e.source) && ids.has(e.target.id??e.target));
}
function buildGraph() {
  const container = document.getElementById('graph-container');
  const W = container.clientWidth, H = container.clientHeight;
  d3.select('#graph-svg').selectAll('*').remove();
  svg = d3.select('#graph-svg').attr('width', W).attr('height', H);
  zoom = d3.zoom().scaleExtent([0.05,4]).on('zoom', e => g.attr('transform', e.transform));
  svg.call(zoom);
  const g = svg.append('g');
  const nodes = getVisibleNodes(), edges = getVisibleEdges(nodes);
  const simNodes = nodes.map(n => ({...n}));
  const nodeById = new Map(simNodes.map(n => [n.id, n]));
  if (simNodes.some(n => n.type==='project')) {
    const degree = {};
    edges.forEach(e => { const s=e.source.id??e.source, t=e.target.id??e.target; degree[s]=(degree[s]||0)+1; degree[t]=(degree[t]||0)+1; });
    const projDegrees = simNodes.filter(n=>n.type==='project').map(n=>degree[n.id]||1);
    const maxDeg=Math.max(...projDegrees,1), minDeg=Math.min(...projDegrees,1);
    simNodes.forEach(n => { if(n.type==='project'){ const d=degree[n.id]||1, t=minDeg===maxDeg?0.5:(d-minDeg)/(maxDeg-minDeg); n._radius=PROJECT_RADIUS_MIN+t*(PROJECT_RADIUS_MAX-PROJECT_RADIUS_MIN); }});
  }
  const simEdges = edges.map(e=>({source:nodeById.get(e.source.id??e.source),target:nodeById.get(e.target.id??e.target)})).filter(e=>e.source&&e.target);
  simulation = d3.forceSimulation(simNodes)
    .force('link', d3.forceLink(simEdges).id(d=>d.id).distance(d=>{ const s=d.source.type,t=d.target.type; if(s==='project'||t==='project')return 120; if(s==='thought'||t==='thought')return 80; return 50; }).strength(0.4))
    .force('charge', d3.forceManyBody().strength(d=>{ if(d.type==='project')return -200-(d._radius||15)*12; if(d.type==='thought')return -120; if(d.type==='session')return -60; return -20; }))
    .force('center', d3.forceCenter(W/2, H/2))
    .force('collision', d3.forceCollide().radius(d=>(d._radius||NODE_RADIUS[d.type]||4)+4))
    .alphaDecay(0.02);
  linkSel = g.append('g').selectAll('line').data(simEdges).join('line').attr('class','link');
  nodeSel = g.append('g').selectAll('g.node').data(simNodes).join('g').attr('class','node')
    .call(d3.drag()
      .on('start',(event,d)=>{ if(!event.active)simulation.alphaTarget(0.3).restart(); d.fx=d.x; d.fy=d.y; draggedNode=d; })
      .on('drag', (event,d)=>{ d.fx=event.x; d.fy=event.y; if(event.sourceEvent)updateEdgeVelocity(event.sourceEvent.clientX,event.sourceEvent.clientY); })
      .on('end',  (event,d)=>{ if(!event.active)simulation.alphaTarget(0); d.fx=null; d.fy=null; draggedNode=null; edgeVx=0; edgeVy=0; }))
    .on('click',(event,d)=>{ event.stopPropagation(); selectNode(d); })
    .on('mouseover',(event,d)=>showTooltip(event,d))
    .on('mousemove',(event)=>moveTooltip(event))
    .on('mouseout',hideTooltip);
  nodeSel.append('circle').attr('r',d=>d._radius||NODE_RADIUS[d.type]||4).attr('class',d=>`type-${d.type}`);
  nodeSel.filter(d=>d.type==='project'||d.type==='thought'||d.type==='key_output')
    .append('text').attr('dy',d=>(d._radius||NODE_RADIUS[d.type])+12).attr('text-anchor','middle').text(d=>d.label);
  svg.on('click',()=>{ selectedNode=null; clearHighlight(); closePreview(); });
  simulation.on('tick',()=>{
    linkSel.attr('x1',d=>d.source.x).attr('y1',d=>d.source.y).attr('x2',d=>d.target.x).attr('y2',d=>d.target.y);
    nodeSel.attr('transform',d=>`translate(${d.x},${d.y})`);
  });
}
function selectNode(d){ selectedNode=d; highlightNeighbors(d); openPreview(d); }
function highlightNeighbors(d){
  const ids=new Set([d.id]);
  linkSel.each(e=>{ const s=e.source.id??e.source,t=e.target.id??e.target; if(s===d.id)ids.add(t); if(t===d.id)ids.add(s); });
  nodeSel.classed('dimmed',n=>!ids.has(n.id));
  linkSel.classed('dimmed',e=>{ const s=e.source.id??e.source,t=e.target.id??e.target; return !ids.has(s)||!ids.has(t); });
  linkSel.classed('highlighted',e=>{ const s=e.source.id??e.source,t=e.target.id??e.target; return s===d.id||t===d.id; });
}
function clearHighlight(){ if(nodeSel)nodeSel.classed('dimmed',false); if(linkSel){linkSel.classed('dimmed',false);linkSel.classed('highlighted',false);} }
function openPreview(d){
  const panel=document.getElementById('preview-panel');
  document.getElementById('preview-title').textContent=d.title||d.label;
  const badge=document.getElementById('preview-badge');
  badge.textContent=d.type; badge.style.background=BADGE_COLORS[d.type]+'33'; badge.style.color=BADGE_COLORS[d.type];
  panel.classList.add('open');
  const body=document.getElementById('preview-body');
  if(d.path){
    const ext=d.path.split('.').pop().toLowerCase();
    const binary=['docx','pptx','xlsx','pdf','doc','xls','png','jpg','jpeg','gif','zip'];
    if(binary.includes(ext)){
      body.innerHTML=`<div style="padding:12px;background:var(--surface0);border-radius:8px"><div style="color:var(--subtext);font-size:12px;margin-bottom:6px">Binary file — cannot preview</div><code style="font-size:11px;color:var(--peach);word-break:break-all">${d.path}</code></div>`;
      return;
    }
    if(['html','htm'].includes(ext)){
      body.innerHTML=`<div style="font-size:11px;color:var(--overlay0);margin-bottom:8px">HTML preview</div><iframe src="/api/file?path=${encodeURIComponent(d.path)}&raw=1" style="width:100%;height:calc(100vh - 120px);border:none;border-radius:6px;background:#fff"></iframe>`;
      return;
    }
    if(ext==='canvas'){
      body.innerHTML='<div style="color:var(--overlay0);font-size:12px">Loading…</div>';
      fetch('/api/file?path='+encodeURIComponent(d.path)).then(r=>r.text()).then(text=>{
        let canvas; try{canvas=JSON.parse(text);}catch{body.innerHTML='<p style="color:var(--red)">Invalid canvas JSON</p>';return;}
        const ns=canvas.nodes||[], textNs=ns.filter(n=>n.type==='text'&&n.text), fileNs=ns.filter(n=>n.type==='file'&&n.file), groupNs=ns.filter(n=>n.type==='group');
        let html=`<div style="margin-bottom:12px"><span style="font-size:11px;color:var(--overlay0)">${ns.length} nodes · ${(canvas.edges||[]).length} edges</span></div>`;
        if(groupNs.length) html+=`<h2 style="font-size:13px;margin-bottom:8px;color:var(--mauve)">Groups (${groupNs.length})</h2><ul style="margin-bottom:14px">`+groupNs.map(n=>`<li style="margin:3px 0;font-size:12px;color:var(--subtext)">${n.label||n.id}</li>`).join('')+'</ul>';
        if(fileNs.length) html+=`<h2 style="font-size:13px;margin-bottom:8px;color:var(--blue)">Linked files (${fileNs.length})</h2><ul style="margin-bottom:14px">`+fileNs.map(n=>`<li style="margin:3px 0;font-size:12px;color:var(--peach);word-break:break-all">${n.file}</li>`).join('')+'</ul>';
        if(textNs.length){ html+=`<h2 style="font-size:13px;margin-bottom:8px;color:var(--green)">Text nodes (${textNs.length})</h2>`; textNs.forEach(n=>{const p=n.text.length>200?n.text.slice(0,200)+'…':n.text;html+=`<div style="background:var(--surface0);border-radius:6px;padding:8px 10px;margin-bottom:8px;font-size:12px;white-space:pre-wrap;line-height:1.5">${p.replace(/</g,'&lt;')}</div>`;});}
        body.innerHTML=html;
      }).catch(()=>{body.innerHTML=`<p style="color:var(--red)">Cannot load file.<br><code>${d.path}</code></p>`;});
      return;
    }
    body.innerHTML='<div style="color:var(--overlay0);font-size:12px">Loading…</div>';
    fetch('/api/file?path='+encodeURIComponent(d.path)).then(r=>r.text()).then(text=>{body.innerHTML=marked.parse(text);}).catch(()=>{body.innerHTML=`<p style="color:var(--red)">Cannot load file.<br><code>${d.path}</code></p>`;});
  } else {
    const sessions=allNodes.filter(n=>n.type==='session'&&n.project===d.label);
    const sessionIds=new Set(sessions.map(s=>s.id));
    const s2o={};
    allEdges.forEach(e=>{const s=e.source.id??e.source,t=e.target.id??e.target;if(sessionIds.has(s)&&t.startsWith('key_output:')){if(!s2o[s])s2o[s]=[];const ko=allNodes.find(n=>n.id===t);if(ko)s2o[s].push(ko);}});
    const swo=sessions.filter(s=>s2o[s.id]?.length).sort((a,b)=>a.label.localeCompare(b.label));
    const total=swo.reduce((acc,s)=>acc+s2o[s.id].length,0);
    let html=`<h2 style="margin-bottom:6px">Project: ${d.label}</h2><p style="color:var(--subtext);font-size:12px;margin-bottom:14px">${sessions.length} sessions · ${total} key outputs</p>`;
    if(!swo.length){html+=`<p style="color:var(--overlay0);font-size:12px">No key outputs recorded.</p>`;}
    else{swo.forEach(s=>{const outs=s2o[s.id];html+=`<div style="margin-bottom:12px"><div style="font-size:11px;color:var(--overlay0);margin-bottom:4px">${s.label}</div>`;outs.forEach(ko=>{html+=`<div style="background:var(--surface0);border-radius:6px;padding:6px 10px;margin-bottom:4px;cursor:pointer" data-ko-id="${ko.id}"><span style="font-size:11px;color:#f59e0b;margin-right:6px">⬡</span><span style="font-size:12px;color:var(--text)">${ko.label}</span>${ko.description?`<div style="font-size:11px;color:var(--subtext);margin-top:3px;padding-left:17px">${ko.description}</div>`:''}</div>`;});html+=`</div>`;});}
    body.innerHTML=html;
    body.querySelectorAll('[data-ko-id]').forEach(el=>el.addEventListener('click',()=>{const node=allNodes.find(n=>n.id===el.dataset.koId);if(node)selectNode(node);}));
  }
}
function closePreview(){ document.getElementById('preview-panel').classList.remove('open'); }
document.getElementById('preview-close').addEventListener('click',()=>{ selectedNode=null; clearHighlight(); closePreview(); });
function showTooltip(event,d){ const tt=document.getElementById('tooltip'); document.getElementById('tt-title').textContent=d.label; document.getElementById('tt-sub').textContent=[d.type,d.project?`project: ${d.project}`:'',d.title||''].filter(Boolean).join(' · '); tt.style.display='block'; moveTooltip(event); }
function moveTooltip(event){ const tt=document.getElementById('tooltip'); tt.style.left=(event.clientX+14)+'px'; tt.style.top=(event.clientY-10)+'px'; }
function hideTooltip(){ document.getElementById('tooltip').style.display='none'; }
['project','thought','session','key_output'].forEach(type=>{
  document.getElementById(`f-${type}`).addEventListener('change',e=>{ if(e.target.checked)visibleTypes.add(type); else visibleTypes.delete(type); rebuildGraph(); });
});
document.getElementById('search-box').addEventListener('input',e=>{ searchQuery=e.target.value.trim(); rebuildGraph(); });
function rebuildGraph(){ if(simulation)simulation.stop(); selectedNode=null; closePreview(); buildGraph(); updateStats(); }
function updateStats(){
  const vis=getVisibleNodes(), counts={};
  vis.forEach(n=>counts[n.type]=(counts[n.type]||0)+1);
  document.getElementById('stats').innerHTML=[`<b style="color:var(--text)">${vis.length}</b> nodes visible`,`Projects: ${counts.project||0}`,`Thoughts: ${counts.thought||0}`,`Sessions: ${counts.session||0}`,counts.key_output?`Key outputs: ${counts.key_output}`:''  ].filter(Boolean).join('<br>');
}
setInterval(async()=>{ try{ const res=await fetch('/api/graph'); const data=await res.json(); if(JSON.stringify(data.nodes.length)!==JSON.stringify(allNodes.length)){allNodes=data.nodes;allEdges=data.edges;rebuildGraph();} }catch{} },10000);
window.addEventListener('resize',()=>{ if(simulation)rebuildGraph(); });
const EDGE_ZONE=80, EDGE_SPEED=10;
let edgeVx=0, edgeVy=0, edgeFrameId=null;
function edgeScrollStep(){ if(!draggedNode||(edgeVx===0&&edgeVy===0)){edgeFrameId=null;return;} if(svg&&zoom){zoom.translateBy(svg,edgeVx,edgeVy);const t=d3.zoomTransform(svg.node());draggedNode.fx-=edgeVx/t.k;draggedNode.fy-=edgeVy/t.k;} edgeFrameId=requestAnimationFrame(edgeScrollStep); }
function updateEdgeVelocity(x,y){ if(!draggedNode)return; const W=window.innerWidth,H=window.innerHeight; let vx=0,vy=0; if(x<EDGE_ZONE)vx=+EDGE_SPEED*(1-x/EDGE_ZONE); if(x>W-EDGE_ZONE)vx=-EDGE_SPEED*(1-(W-x)/EDGE_ZONE); if(y<EDGE_ZONE)vy=+EDGE_SPEED*(1-y/EDGE_ZONE); if(y>H-EDGE_ZONE)vy=-EDGE_SPEED*(1-(H-y)/EDGE_ZONE); edgeVx=vx;edgeVy=vy; if((vx!==0||vy!==0)&&!edgeFrameId)edgeFrameId=requestAnimationFrame(edgeScrollStep); }
window.addEventListener('mouseleave',e=>{ if(!draggedNode)return; const W=window.innerWidth,H=window.innerHeight; let vx=0,vy=0; if(e.clientX<=0)vx=+EDGE_SPEED; else if(e.clientX>=W)vx=-EDGE_SPEED; if(e.clientY<=0)vy=+EDGE_SPEED; else if(e.clientY>=H)vy=-EDGE_SPEED; edgeVx=vx;edgeVy=vy; if((vx!==0||vy!==0)&&!edgeFrameId)edgeFrameId=requestAnimationFrame(edgeScrollStep); });
window.addEventListener('mouseenter',()=>{ edgeVx=0;edgeVy=0; });
loadGraph();
</script>
</body>
</html>
```

---

## Step 3 — Start the server

```bash
cd "GRAPH_DIR"
python server.py
```

The server auto-opens the browser at `http://localhost:PORT`.

---

## Step 4 — Verify

After starting, confirm in the terminal output:
```
[server] Graph ready: N nodes, E edges
```
And the browser shows the D3 force graph with coloured nodes.

---

## Node types & colours

| Type | Colour | Description |
|:---|:---|:---|
| 🟠 Project | Peach | Inferred from `project:` tag in session files |
| 🔵 Thought | Blue | One per `thoughts_YYYY-MM-DD.md` file |
| ⬜ Session | Grey | Each `session N.md` file |
| 🟡 Key Output | Amber | `🎯 Main Output:` line in thought files |

---

## Prerequisites

- Python 3 (stdlib only, no pip installs needed)
- Folder structure with `thought-recording/thoughts_*.md` and/or `focus-sessions/**/session *.md`

---

## Refresh graph

The server auto-detects file changes (mtime-based cache). Just save a new `.md` file and refresh the browser — no restart needed.
