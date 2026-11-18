# Node.js Jenkins Pipeline with SemVer

A learning / demo repo that combines three ideas:

1. A **simple Create React App** UI (the classic “Welcome to React” page)
2. A **Jenkins Declarative Pipeline** that installs, tests, builds, and temporarily serves the app
3. **Semantic Versioning** via [`semantic-release`](https://github.com/semantic-release/semantic-release), so releases can be cut automatically from Conventional Commits

It started from the official Jenkins tutorial
[Build a Node.js and React app with npm](https://www.jenkins.io/doc/tutorials/build-a-node-js-and-react-app-with-npm/)
and was extended to practice CI/CD + versioning.

## About / purpose

| Goal | What this repo shows |
|---|---|
| **CI with Jenkins** | How a `Jenkinsfile` runs Build → Test → Deliver on a Node Docker agent |
| **Manual release gate** | The Deliver stage pauses for a human “Proceed” before shutting the preview down |
| **SemVer automation** | How `semantic-release` can turn commit messages into version tags / GitHub releases |
| **App under test** | A minimal React app with a Jest smoke test (`src/App.test.js`) |

This is a **training repo**, not a production product. The React app itself is intentionally tiny so the focus stays on the pipeline and versioning.

## How the pieces fit

```text
  Developer push / PR
           │
           ├──────────────► GitHub Actions (optional experiment workflows)
           │
           ▼
     Jenkins Pipeline (jenkins/Jenkinsfile)
           │
           ├─ Build   npm install
           ├─ Test    jenkins/scripts/test.sh  →  npm test
           └─ Deliver jenkins/scripts/deliver.sh
                      ├─ npm run build
                      ├─ npm start (preview on :3000)
                      ├─ [manual] Proceed?
                      └─ jenkins/scripts/kill.sh

  Separately (when configured):
     npm run semantic-release
           └─ reads Conventional Commits → bumps version → Git tag / GitHub Release
```

### Jenkins pipeline stages

Defined in [`jenkins/Jenkinsfile`](jenkins/Jenkinsfile):

| Stage | What runs | Why |
|---|---|---|
| **Build** | `npm install` | Install app + tool dependencies inside `node:lts-buster-slim` |
| **Test** | `./jenkins/scripts/test.sh` | Runs Jest with `CI=true` so tests don’t hang in watch mode |
| **Deliver** | `deliver.sh` → **manual input** → `kill.sh` | Production build, start a short-lived preview on port **3000**, wait for you to click Proceed, then stop the process |

The agent maps host port 3000 into the container (`args '-p 3000:3000'`) so you can open the preview while the job waits.

### Semantic release (SemVer)

- `package.json` sets `"version": "0.0.0-development"` (normal for semantic-release).
- Script: `npm run semantic-release`.
- Intended flow: commits like `feat: …`, `fix: …`, `BREAKING CHANGE: …` drive **minor / patch / major** bumps.
- To use it for real you need a CI token with permission to push tags/releases to this GitHub repo, and usually a release config (defaults work for `master`/`main` + Conventional Commits).

> The Jenkinsfile in this repo does **not** run `semantic-release` yet. Versioning is available as an npm script you can wire into Jenkins or GitHub Actions when you want automated releases.

### GitHub Actions workflows (experiments)

Under `.github/workflows/`:

| Workflow | Trigger | Behaviour |
|---|---|---|
| `main.yml` (“Default pass”) | PR opened | Passes when the calendar month is **even** |
| `workflow.yml` (“Check Month”) | PR opened | **Fails** when the calendar month is **odd** |

These are **demo PR checks** for learning branch protection / required status quirks — they are not part of the Node/React build. Treat them as optional experiments.

## Project layout

```text
.
├── jenkins/
│   ├── Jenkinsfile          # Declarative pipeline
│   └── scripts/
│       ├── test.sh          # npm test
│       ├── deliver.sh       # npm run build + npm start
│       └── kill.sh          # stop preview via .pidfile
├── src/                     # React app + App.test.js
├── public/                  # CRA static assets
├── .github/workflows/       # Experimental month-based PR checks
└── package.json             # react-scripts + semantic-release
```

## Local run

Requires **Node.js 16+** (CRA 4 / react-scripts 4 era; newer Node may need legacy OpenSSL flags).

```bash
git clone https://github.com/ironman-gampala/nodejs-jenkinspipeline_with_semver.git
cd nodejs-jenkinspipeline_with_semver
npm install
npm start          # http://localhost:3000
npm test           # Jest
npm run build      # production bundle in build/
```

Install release tooling (already listed under `devDependencies`):

```bash
npm install
npx semantic-release --dry-run   # safe local preview if you have a GitHub token configured
```

## Run with Jenkins

1. Install Jenkins with the **Pipeline** and **Docker** plugins (or use a Jenkins agent that can run Docker).
2. Create a **Pipeline** job → “Pipeline script from SCM” → point at this repo → script path `jenkins/Jenkinsfile`.
3. Ensure the Jenkins agent can pull `node:lts-buster-slim` and bind port **3000**.
4. Run the job:
   - Watch **Build** and **Test** succeed.
   - Open `http://<jenkins-host>:3000` during **Deliver**.
   - Click **Proceed** when finished; **kill.sh** stops the preview.

Official walkthrough this repo is based on:
https://www.jenkins.io/doc/tutorials/build-a-node-js-and-react-app-with-npm/

## SemVer cheat sheet (Conventional Commits)

| Commit message | Version bump |
|---|---|
| `fix: correct button label` | patch (`1.0.0` → `1.0.1`) |
| `feat: add about page` | minor (`1.0.0` → `1.1.0`) |
| `feat!: drop Node 12` / `BREAKING CHANGE:` footer | major (`1.0.0` → `2.0.0`) |

## Repo

- **Owner:** [ironman-gampala](https://github.com/ironman-gampala)
- **Default branch:** `master`
- **GitHub:** https://github.com/ironman-gampala/nodejs-jenkinspipeline_with_semver
