# Frontend System Design

This repository contains frontend artifacts and notes for system design exercises and small demo pages.

Status: initial stub — contains static HTML pages and supporting notes.

## Contents

- `1-2-days.html` — a static HTML page included in the workspace (example/system-design notes).
- `README.md` — this file.

(As the project grows, we'll add folders for prototypes, diagrams, components, and experiments.)

## Goals

- Keep a lightweight working repo for frontend system design experiments.
- Collect small runnable examples (static pages, interactive demos).
- Document architecture notes, trade-offs, and system diagrams.

## Getting started

You can open the static HTML files directly in a browser or serve them over a local HTTP server (recommended for testing relative assets).

Prerequisites
- A modern browser (Chrome, Firefox, Safari)
- (Optional) Node.js/npm if you want to install a static server like `http-server` or `serve`.

Open directly
- In your file manager or editor, open `1-2-days.html`.

Serve locally (recommended)
- Using Python 3 (macOS):

```bash
cd /Users/surajy/WebstormProjects/system_design_frontend
python3 -m http.server 8000
# then open http://localhost:8000/1-2-days.html
```

- Or using npm `http-server`:

```bash
cd /Users/surajy/WebstormProjects/system_design_frontend
npm install --global http-server
http-server -p 8000
# then open http://localhost:8000/1-2-days.html
```

## Suggested repository structure

As this repo grows, consider organizing like:

```
/ (repo root)
  README.md
  /examples      # runnable demo pages and micro-apps
  /notes         # markdown notes and architecture docs
  /diagrams      # images, diagrams (draw.io, svg)
  /components    # reusable UI components (if you add a build tool)
```

## Contributing

- Add small, self-contained examples.
- Include a short README per example explaining what the page/demo demonstrates and any run steps.
- Prefer simple, dependency-free demos (vanilla JS/HTML/CSS) or include a `package.json` if you add toolchain/deps.

## Next steps (ideas)

- Add a `package.json` and simple dev script (e.g., `npm run start`) to standardize serving examples.
- Add a `CONTRIBUTING.md` and issue/PR templates.
- Add a few system-design case studies: caching, load balancing, client-side state management, offline-first strategies.

## License

This repository is provided under the MIT license — add a `LICENSE` file if you want an explicit copy.

## Contact

If you'd like me to expand this README (add scripts, tests, or a sample `package.json`), tell me what you want and I will add it.

