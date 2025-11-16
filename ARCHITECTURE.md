# Portfolio Hub Architecture

This document provides a visual overview of how the portfolio hub system works.

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub Profile                          │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Pinned     │  │  Pinned     │  │  Pinned     │        │
│  │  Repo 1     │  │  Repo 2     │  │  Repo 3     │  ...   │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
                           ↓
                   (GraphQL API Query)
                           ↓
┌─────────────────────────────────────────────────────────────┐
│           GitHub Actions Workflow                           │
│      (.github/workflows/deploy-and-update.yml)              │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ Job 1: update-projects                                │ │
│  │  1. Fetch pinned repos via GraphQL                    │ │
│  │  2. Download thumbnails from Open Graph images        │ │
│  │  3. Update projects.json                              │ │
│  │  4. Commit changes                                    │ │
│  └───────────────────────────────────────────────────────┘ │
│                           ↓                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ Job 2: build-and-deploy-projects                      │ │
│  │  1. Create dist/ directory                            │ │
│  │  2. Copy main site files                              │ │
│  │  3. For each project:                                 │ │
│  │     • Clone repository                                │ │
│  │     • If Node.js: npm install && npm run build        │ │
│  │     • If Static: copy HTML/CSS/JS                     │ │
│  │     • Copy to dist/<project-name>/                    │ │
│  │  4. Upload artifact                                   │ │
│  └───────────────────────────────────────────────────────┘ │
│                           ↓                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ Job 3: deploy                                         │ │
│  │  1. Deploy artifact to GitHub Pages                   │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                           ↓
                   (GitHub Pages Deployment)
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                  GitHub Pages Server                        │
│           https://projects.brianwalker.dev                  │
│                                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │ /                  → Main portfolio page (index)   │    │
│  │ /projects.json     → Project metadata             │    │
│  │ /assets/           → Main site CSS/JS/images      │    │
│  │                                                    │    │
│  │ /responsive-web-page-html-css/                    │    │
│  │    → Featured project 1 (live demo)               │    │
│  │                                                    │    │
│  │ /frontend-web-application/                        │    │
│  │    → Featured project 2 (live demo)               │    │
│  │                                                    │    │
│  │ /game-show-web-app/                               │    │
│  │    → Featured project 3 (live demo)               │    │
│  │                                                    │    │
│  │ ... (more projects)                                │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow

### 1. Project Discovery Flow

```
User's GitHub Profile
  └─> Pinned Repositories (up to 6)
       └─> GraphQL API Query
            └─> fetch-pinned-repos.sh
                 └─> projects.json
                      └─> Contains:
                           • name
                           • description
                           • updated timestamp
                           • url (GitHub repo)
                           • homepage (demo URL)
                           • thumbnail path
                           • openGraphImageUrl
```

### 2. Build & Deploy Flow

```
projects.json
  └─> For each project:
       ├─> Clone repository
       │    └─> Check for package.json
       │         ├─> YES: Node.js project
       │         │    └─> npm install
       │         │         └─> npm run build
       │         │              └─> Copy from dist/build/public/
       │         │                   └─> dist/<project-name>/
       │         │
       │         └─> NO: Static project
       │              └─> Copy *.html, *.css, *.js, assets/
       │                   └─> dist/<project-name>/
       │
       └─> Also copy:
            ├─> index.html → dist/
            ├─> projects.json → dist/
            └─> assets/ → dist/assets/
```

### 3. URL Resolution Flow

```
User clicks "Live Demo" button
  └─> projects.json lookup
       ├─> homepage = "https://external.com/project"
       │    └─> External URL → Redirect to external site
       │
       ├─> homepage = "/<project-name>/"
       │    └─> Relative URL → https://projects.brianwalker.dev/<project-name>/
       │
       └─> homepage = "https://brianwalkerdev.github.io/<project>/"
            └─> GitHub Pages URL → Redirect to other GitHub Pages site
```

## Component Interactions

### Main Portfolio Page (index.html)

```
┌──────────────────────────────────────┐
│        Main Portfolio Page           │
│                                      │
│  1. Loads projects.json              │
│  2. Renders project cards            │
│  3. Each card has:                   │
│     • Thumbnail image                │
│     • Project name                   │
│     • Description                    │
│     • "Live Demo" button            │
│     • "View Code" button            │
│                                      │
│  4. Features:                        │
│     • Search/filter projects         │
│     • Sort by date/name              │
│     • Dark/light mode toggle         │
│     • Theme color selector           │
│     • Responsive design              │
└──────────────────────────────────────┘
```

### Project JSON Schema

```json
{
  "name": "project-name",           // Repository name (used for URL path)
  "description": "...",             // From repository description
  "updated": "2025-11-16T...",      // Last update timestamp
  "url": "https://github.com/...", // GitHub repository URL
  "homepage": "...",                // Live demo URL (3 options)
  "thumbnail": "assets/img/...",    // Local thumbnail path
  "openGraphImageUrl": "https://...", // GitHub OG image URL
  "usesCustomOpenGraphImage": true  // Whether has custom image
}
```

### Workflow Triggers

```
Trigger Events:
  ├─> Push to main branch
  │    └─> Deploys immediately after merge
  │
  ├─> Weekly schedule (cron)
  │    └─> Sundays at 00:00 UTC
  │         └─> Keeps projects up-to-date
  │
  └─> Manual workflow dispatch
       └─> Actions tab → Run workflow button
            └─> On-demand deployment
```

## Directory Structure After Deployment

```
dist/                                    # Deployed to GitHub Pages
├── index.html                          # Main portfolio hub
├── projects.json                       # Project metadata
├── assets/                             # Main site assets
│   ├── css/
│   │   └── style.css
│   ├── js/
│   │   └── main.js
│   └── img/
│       ├── frontend-web-application.png
│       ├── game-show-web-app.png
│       └── ... (more thumbnails)
│
├── responsive-web-page-html-css/      # Featured project 1
│   ├── index.html
│   ├── css/
│   │   ├── styles.css
│   │   └── normalize.css
│   └── images/
│       └── ...
│
├── frontend-web-application/          # Featured project 2
│   ├── index.html
│   ├── css/
│   ├── js/
│   ├── images/
│   └── svgs/
│
├── game-show-web-app/                 # Featured project 3
│   ├── index.html
│   ├── css/
│   └── js/
│
├── interactive-image-gallery-js/      # Featured project 4
│   ├── index.html
│   ├── css/
│   ├── js/
│   └── photos/
│
├── responsive-web-page-advanced-layouts/ # Featured project 5
│   ├── index.html
│   └── css/
│
└── modern-web-page-sass/              # Featured project 6
    ├── index.html
    └── css/
```

## Live URL Mapping

| Project | GitHub Repo | Live Demo URL |
|---------|-------------|---------------|
| Main Portfolio | brianwalkerdev/projects.brianwalker.dev | https://projects.brianwalker.dev/ |
| Project 1 | brianwalkerdev/responsive-web-page-html-css | https://projects.brianwalker.dev/responsive-web-page-html-css/ |
| Project 2 | brianwalkerdev/frontend-web-application | https://projects.brianwalker.dev/frontend-web-application/ |
| Project 3 | brianwalkerdev/game-show-web-app | https://projects.brianwalker.dev/game-show-web-app/ |
| Project 4 | brianwalkerdev/interactive-image-gallery-js | https://projects.brianwalker.dev/interactive-image-gallery-js/ |
| Project 5 | brianwalkerdev/responsive-web-page-advanced-layouts | https://projects.brianwalker.dev/responsive-web-page-advanced-layouts/ |
| Project 6 | brianwalkerdev/modern-web-page-sass | https://projects.brianwalker.dev/modern-web-page-sass/ |

## Workflow Execution Timeline

```
Time: 00:00 UTC (Scheduled) or Immediately (Push/Manual)
│
├─ 00:00:00 - Workflow triggered
├─ 00:00:05 - Job 1 starts: update-projects
│   ├─ Fetch pinned repos (5s)
│   ├─ Download thumbnails (10s)
│   └─ Commit changes (2s)
├─ 00:00:22 - Job 1 completes
│
├─ 00:00:25 - Job 2 starts: build-and-deploy-projects
│   ├─ Setup environment (5s)
│   ├─ Clone and build project 1 (20s)
│   ├─ Clone and build project 2 (25s)
│   ├─ Clone and build project 3 (15s)
│   ├─ Clone and build project 4 (18s)
│   ├─ Clone and build project 5 (12s)
│   ├─ Clone and build project 6 (22s)
│   └─ Upload artifact (5s)
├─ 00:02:47 - Job 2 completes
│
├─ 00:02:50 - Job 3 starts: deploy
│   └─ Deploy to GitHub Pages (15s)
├─ 00:03:05 - Job 3 completes
│
└─ 00:03:10 - Site live at https://projects.brianwalker.dev

Total time: ~3 minutes
```

## Key Files and Their Roles

| File | Purpose | Modified By |
|------|---------|-------------|
| `index.html` | Main portfolio page | Developer (manual) |
| `projects.json` | Project metadata | Workflow (automatic) |
| `assets/css/style.css` | Main site styles | Developer (manual) |
| `assets/js/main.js` | Main site functionality | Developer (manual) |
| `assets/img/*.png` | Project thumbnails | Workflow (automatic) |
| `.github/workflows/deploy-and-update.yml` | Deployment automation | Developer (manual) |
| `.github/scripts/fetch-pinned-repos.sh` | Fetch pinned repos | Developer (manual) |
| `.github/scripts/download-thumbnails.sh` | Download thumbnails | Developer (manual) |
| `CNAME` | Custom domain config | Developer (manual) |
| `.nojekyll` | Disable Jekyll processing | Workflow (automatic) |

## Security & Permissions

```
GitHub Token Permissions:
├─ contents: write
│   └─ Allows committing projects.json updates
│
├─ pages: write
│   └─ Allows deploying to GitHub Pages
│
└─ id-token: write
    └─ Required for Pages deployment authentication
```

## Summary

This architecture provides:
- ✅ **Automatic Discovery**: Pinned repos automatically detected
- ✅ **Automatic Building**: Projects built without manual intervention
- ✅ **Automatic Deployment**: One-click deployment via workflow
- ✅ **Automatic Updates**: Weekly refresh keeps content current
- ✅ **Zero Maintenance**: System runs entirely via GitHub Actions

The entire system is **fully automated** and requires no manual intervention after initial setup.
