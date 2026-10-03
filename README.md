# AppCode — React ToDo App with a CI/GitOps Pipeline

A simple ToDo list web application built with **React** and **React-Bootstrap**, packaged as a multi-stage **Docker** image served by **Nginx**, and delivered through a **CircleCI** pipeline that follows a **GitOps** model: CI builds and publishes the image, then updates a separate manifest repository that the cluster deploys from.

This repository holds the **application code and CI definition**. Kubernetes manifests live in a separate repo ([`kube_manifest`](https://github.com/pavitrajena14/kube_manifest)).

---

## Table of Contents

1. [Features](#features)
2. [Tech Stack](#tech-stack)
3. [Repository Structure](#repository-structure)
4. [Run Locally](#run-locally)
5. [Run with Docker](#run-with-docker)
6. [How the Dockerfile Works](#how-the-dockerfile-works)
7. [CI/CD Pipeline (CircleCI)](#cicd-pipeline-circleci)
8. [GitOps Flow](#gitops-flow)
9. [Required CircleCI Environment Variables](#required-circleci-environment-variables)
10. [Application Design Notes](#application-design-notes)
11. [Known Limitations and Improvements](#known-limitations-and-improvements)

---

## Features

- Add a ToDo item (empty input is ignored)
- Edit an existing item (via a prompt dialog; blank edits are ignored)
- Delete an item
- Responsive Bootstrap layout
- State is held in memory (items reset on page refresh)

## Tech Stack

| Layer | Technology |
|---|---|
| UI | React 18, React-Bootstrap 2, Bootstrap 5 |
| Build tooling | react-scripts 5 (Create React App) |
| Testing libs | React Testing Library, jest-dom, user-event |
| Container | Docker multi-stage build (`node:18-alpine` → `nginx`) |
| CI | CircleCI 2.1 |
| Registry | Docker Hub (`pavijena14/todo-app`) |
| Deployment model | GitOps via a separate manifest repository |

## Repository Structure

```
AppCode/
├── .circleci/
│   └── config.yml        # CI pipeline: build/push image, update manifest repo
├── public/               # Static assets and index.html template
├── src/
│   ├── App.js            # Main ToDo component (add / edit / delete)
│   └── ...               # React entry point and supporting files
├── Dockerfile            # Multi-stage build: compile React, serve with Nginx
├── package.json          # Dependencies and npm scripts
└── package-lock.json
```

## Run Locally

**Prerequisites:** Node.js 18+ and npm.

```bash
git clone https://github.com/pavitrajena14/AppCode.git
cd AppCode
npm install
npm start
```

The dev server runs at <http://localhost:3000> with hot reload.

### Available npm scripts

| Script | Purpose |
|---|---|
| `npm start` | Start the development server |
| `npm run build` | Create an optimized production build in `build/` |
| `npm test` | Run tests in watch mode |
| `npm run eject` | Eject from Create React App (irreversible) |

## Run with Docker

```bash
# Build the image
docker build -t todo-app:local .

# Run it (Nginx listens on port 80 inside the container)
docker run --rm -p 8080:80 todo-app:local
```

Open <http://localhost:8080>.

## How the Dockerfile Works

The image uses a two-stage build to keep the final image small:

1. **`installer` stage (`node:18-alpine`)**
   - Copies `package*.json` first so dependency installation is cached unless dependencies change
   - Runs `npm install`, copies the source, then runs `npm run build` to produce static files in `/app/build`
2. **`deployer` stage (`nginx`)**
   - Copies only the compiled `build/` output into `/usr/share/nginx/html`
   - Node, `node_modules` and source code are not included in the final image

## CI/CD Pipeline (CircleCI)

Defined in `.circleci/config.yml` as a single workflow named **`GitOpsflow`** with two sequential jobs.

### Job 1: `build_and_push`

- Runs on `cimg/node:20.3.1` with a remote Docker engine (`setup_remote_docker`)
- Tags the image as `build-<CIRCLE_BUILD_NUM>`
- Builds the image, logs in to Docker Hub, and pushes `pavijena14/todo-app:build-<N>`

### Job 2: `Update_manifest` (runs after `build_and_push` succeeds)

- Clones the `kube_manifest` repository
- Computes the image tag from the current build number minus one, because `Update_manifest` is numbered right after `build_and_push` in the same workflow run
- Uses `sed` to replace the `build-*` image tag in `manifest/deployment.yaml`
- Commits and pushes the change to the `main` branch of `kube_manifest`

```
 push to AppCode ──► build_and_push ──► Docker Hub (todo-app:build-N)
                           │
                           ▼
                    Update_manifest ──► commit new tag to kube_manifest
```

## GitOps Flow

1. A developer pushes code to this repository.
2. CircleCI builds and publishes a uniquely tagged Docker image.
3. CircleCI commits the new tag to `kube_manifest/manifest/deployment.yaml`.
4. A GitOps controller (for example, ArgoCD) watching `kube_manifest` detects the change and syncs the cluster to the new image.

CI never talks to the cluster directly. Git is the single source of truth for what is deployed, which gives you an audit trail and simple rollbacks (revert the manifest commit).

## Required CircleCI Environment Variables

Set these in **Project Settings → Environment Variables**:

| Variable | Used for |
|---|---|
| `DOCKER_USERNAME` | Docker Hub login |
| `DOCKER_PASSWORD` | Docker Hub password or access token |
| `GITHUB_PERSONAL_TOKEN` | Personal access token with push rights to `kube_manifest` |

Never commit these values to the repository.

## Application Design Notes

- `App.js` is a class component. State is `{ userInput, list }`.
- Each item is `{ id, value }`, where `id` comes from `Math.random()` and is used for deletion.
- Edit uses `window.prompt` and updates the item by its index in the list.
- Styling comes from Bootstrap via `react-bootstrap` components (`Container`, `Row`, `Col`, `InputGroup`, `ListGroup`, `Button`).

## Known Limitations and Improvements

- **No persistence:** items are lost on refresh. Consider `localStorage` or a backend API.
- **Item IDs:** replace `Math.random()` with `crypto.randomUUID()` or a counter, and use the id as the React `key` instead of the array index.
- **Edit UX:** replace `prompt()` with inline editing or a modal.
- **Tests:** add component tests with React Testing Library and run `npm test -- --watchAll=false` as a CI step before building the image.
- **Dockerfile:**
  - Pin the Nginx version instead of `latest`
  - Use `npm ci` for reproducible installs
  - Add a `.dockerignore` (`node_modules`, `build`, `.git`)
  - Align the Node version between the Dockerfile (18) and CircleCI (20)
- **Pipeline:**
  - Derive the tag from a shared variable or the Git SHA rather than `CIRCLE_BUILD_NUM - 1`, which is fragile if job numbering changes
  - Run `Update_manifest` only on the `main` branch via a branch filter
  - Add an image vulnerability scan (for example, Trivy) before pushing

## License

No license has been specified for this repository yet. Add a `LICENSE` file if you intend others to reuse the code.
