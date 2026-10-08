---
name: aws-project-bootstrap
description: Generate AWS project scaffolding on demand, including application code, Dockerfile, Terraform, and draw.io architecture diagrams using native draw.io AWS4 stencils (no local icon assets).
metadata:
  version: "1.0"
---

# AWS Project Bootstrap

Use this skill when the user asks for a new AWS project under aws_projects.

## Goal

Generate a complete, runnable project scaffold that usually includes:

- application code for the AWS workload
- a Dockerfile when the project needs a containerized runtime
- Terraform for infrastructure
- a draw.io architecture diagram (`.drawio` source + rendered `.svg`/`.png`)
- a README with realistic run, test, and cleanup steps

## Preferred structure

- Keep the project focused on one coherent AWS use case.
- Prefer Terraform for infrastructure unless the user asks for another IaC tool.
- Use a Python Lambda when the project needs serverless processing.
- Add only the files needed to make the project look and behave real.
- Keep naming professional and project-like, not lab-like.

## Diagram guidance

- All architecture diagrams are built exclusively with draw.io's native AWS4 stencils. Do not use any local icon image assets — there is no `icons/` folder in this workspace.
- Use the draw.io pipeline below for every diagram, even quick ones.
- Show the real project flow clearly, with one focal path and minimal clutter.

## draw.io diagram generation (drawpyo)

Produces a `.drawio` source file plus a rendered `.svg`/`.png` that can be embedded directly in the README.

1. **Install drawpyo in an isolated venv** — the sandbox's system Python is externally-managed (PEP 668), so `pip install` at the system level fails. Create a project-local venv instead:
   ```bash
   python3 -m venv .diagram-venv
   .diagram-venv/bin/pip install drawpyo
   ```
   Add `.diagram-venv/` to `.gitignore`.

2. **Write `scripts/generate_diagram.py`** using drawpyo's API:
   - `file = drawpyo.File(); file.file_name = "architecture.drawio"; file.file_path = "docs"`
   - `page = drawpyo.Page(file=file)`
   - Nodes: `obj = drawpyo.diagram.Object(page=page, value="Label"); obj.position = (x, y); obj.geometry.width = w; obj.geometry.height = h; obj.apply_style_string(style)`
   - Edges: `edge = drawpyo.diagram.Edge(page=page, source=obj1, target=obj2, label="optional")`, then set plain attributes — `edge.waypoints = "orthogonal"`, `edge.endArrow = "block"`, `edge.startArrow = "none"`, `edge.strokeColor = "#545B64"`. Do **not** pass an edge style string to `apply_style_string()`: `Edge.endFill`/`startFill` are read-only properties with no setter and raise `AttributeError`.
   - Finish with `file.write()`.

3. **Use AWS4 style strings for every AWS icon** — these ship with every draw.io renderer (desktop, web, VS Code, GitHub's native `.drawio` viewer), so no external icon library needs to be downloaded, registered, or stored locally:
   ```
   sketch=0;outlineConnect=0;fontColor=#232F3E;gradientColor=none;fillColor=<category-color>;strokeColor=none;dashed=0;verticalLabelPosition=bottom;verticalAlign=top;align=center;html=1;fontSize=11;fontStyle=0;aspect=fixed;pointerEvents=1;shape=mxgraph.aws4.resourceIcon;resIcon=mxgraph.aws4.<icon-name>;
   ```
   - **Always use `resourceIcon`/`resIcon=`, not `productIcon`/`prIcon=`.** The `productIcon` style renders as a blank colored box (no glyph) for several services (S3, SNS, SQS, Lambda) when rasterized; `resourceIcon` renders the glyph reliably for all of them.
   - Confirmed working `resIcon` names: `bucket` (S3), `sns`, `sqs`, `lambda_function` (not `lambda`), `dynamodb`, `api_gateway`, `rekognition_2` (not `rekognition`), `machine_learning` (generic AI/ML), `client` (generic user icon), `security_group` (Security, Identity & Compliance).
   - Typical AWS4 category fill colors: Storage `#7AA116`, Application Integration `#E7157B`, Compute `#ED7100`, Database `#C925D1`, Artificial Intelligence `#01A88D`, Security/Identity/Compliance `#DD344C`, generic/neutral `#232F3E`.
   - For security/network boundary groupings (e.g. a VPC Security Group wrapping other resources), use the **group container** style instead of a plain resource icon, so the box itself carries the glyph and a dashed border:
     ```
     shape=mxgraph.aws4.group;grIcon=mxgraph.aws4.group_security_group;strokeColor=#DD344C;fillColor=none;verticalAlign=top;align=left;spacingLeft=30;fontColor=#DD344C;dashed=1;
     ```
   - For non-AWS services (e.g. Hugging Face, a generic client), use a plain rounded rect style instead of an AWS stencil: `rounded=1;whiteSpace=wrap;html=1;fillColor=#FFF2CC;strokeColor=#D6B656;fontSize=10;fontColor=#7F6000;align=center;`.

4. **Rasterize the `.drawio` file headlessly with Docker** so the diagram can be embedded inline in the README (drawpyo itself only writes XML — it cannot export images):
   ```bash
   docker run --rm -v "$PWD/docs":/data -w /data rlespinasse/drawio-export -f svg -o . --output-mode relative --remove-page-suffix .
   docker run --rm -v "$PWD/docs":/data -w /data rlespinasse/drawio-export -f png -o . --output-mode relative --remove-page-suffix -t .
   ```
   - Exported files land owned by `root` but are world-readable, so `git add`/commit works without `chown`.
   - `-t` / `--transparent` makes the PNG background transparent. Omit it entirely for an opaque white PNG.
   - The exported SVG has a transparent background by default, which looks broken on GitHub's white README background. Force a white background rect right after the opening `<g>` tag:
     ```python
     import re
     with open("docs/architecture.svg") as f:
         content = f.read()
     w, h = re.search(r'viewBox="0 0 (\d+) (\d+)"', content).groups()
     bg_rect = f'<rect x="0" y="0" width="{w}" height="{h}" fill="#ffffff"/>'
     content = content.replace("<defs/><g>", f"<defs/><g>{bg_rect}", 1)
     with open("docs/architecture.svg", "w") as f:
         f.write(content)
     ```

5. **Commit both artifacts and reference them from the README**:
   - Keep `docs/architecture.drawio` as the editable source (GitHub renders `.drawio` files natively when opened directly in the repo UI, and it can be reopened in draw.io desktop / app.diagrams.net).
   - Embed the rasterized SVG inline so it actually shows in the README body: `![Architecture diagram](docs/architecture.svg)`.
   - Document the regeneration commands (venv install + script + docker export + white-background patch) in the README so the diagram can be rebuilt after future architecture changes.

## Writing guidance

- Write documentation in English unless the user asks otherwise.
- Name the project like a real portfolio project, not like a lab exercise.
- Prefer short, practical sections: overview, architecture, deploy, test, cleanup.
- When the user asks for "other AWS projects," interpret that as a request to generate a complete starter project, not just a prose explanation.

## Validation checklist

- Terraform should validate.
- The generated app code should be syntactically correct.
- The Dockerfile should build if one is included.
- The diagram should be a draw.io file built only from AWS4 stencils, with no references to local icon images.
- The README or docs should point to real run commands, not just describe them.
