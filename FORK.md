# Fork Workflow (felores/thepopebot)

This repo is a fork of `stephengpope/thepopebot`. Two types of changes are managed here.

## Remotes

| Remote | Repo | Purpose |
|--------|------|---------|
| `origin` | `felores/thepopebot` | Our fork — push here |
| `upstream` | `stephengpope/thepopebot` | Original repo — pull updates, target for PRs |

## Branch Strategy

| Branch | Purpose | PR target |
|--------|---------|-----------|
| `main` | Stays in sync with upstream. Base for contribution branches. | — |
| `feat/*`, `fix/*` | Upstream contributions (e.g., `feat/update-models`, `fix/chat-input-focus`) | `upstream/main` |
| `custom` | Deployment branch. Merges `main` + pending feature branches + personal changes. Never PR'd upstream. | — |

## How `custom` Works With Upstream PRs

`custom` merges in feature branches while they're pending upstream review. When upstream accepts a PR:

1. Those changes land in `upstream/main`
2. You sync `main`: `git fetch upstream && git merge upstream/main`
3. You rebase `custom` on `main`: `git checkout custom && git rebase main`
4. Git sees the PR commits are already applied and **skips them automatically**
5. Only custom-only commits (branding, Docker infra) remain

**No duplication, no conflicts, no double-fixes.** This is standard open source fork workflow.

## Upstream Contributions

**RULES:**
1. Never send PRs before testing the modifications on our dev-bots instance first. Build a patched image, deploy it, verify the fix/feature works.
2. **Never create upstream PRs without explicit user approval.** Present the diff and PR description to the user first. Only create the PR after they confirm.

```bash
git fetch upstream && git merge upstream/main   # sync main first
git checkout -b feat/my-feature main            # branch off main
# make changes
# TEST ON DEV-BOTS FIRST (see "Testing Changes" section below)
git push origin feat/my-feature                 # push to fork
gh pr create -R stephengpope/thepopebot         # PR to upstream
```

After creating the feature branch, merge it into `custom` for testing:

```bash
git checkout custom && git merge feat/my-feature
# rebuild + deploy (see "Testing Changes")
```

## Personal Customizations

Changes that should never go upstream (branding, deployment config):

```bash
git checkout custom
# make changes, commit
git push origin custom --force-with-lease
```

## Syncing With Upstream

```bash
git checkout main && git fetch upstream && git merge upstream/main
git checkout custom && git rebase main
# Force push since rebase rewrites history (custom is our private branch)
git push origin custom --force-with-lease
```

## Testing Changes on dev-bots (bot.markenetica.com)

The deployed instance (`felores/dev-bots`) runs Docker images. To test fork changes:

### Build on server (fast — native x86, ~1 min)

```bash
ssh neo4j "cd /tmp && rm -rf thepopebot-build && git clone --depth 1 -b custom https://github.com/felores/thepopebot.git thepopebot-build"
ssh neo4j "cd /tmp/thepopebot-build && docker build -t ghcr.io/felores/thepopebot:event-handler-patched -f docker/event-handler/Dockerfile.patched ."
ssh neo4j "docker push ghcr.io/felores/thepopebot:event-handler-patched"
```

### Build locally (slow — cross-compile amd64 on ARM Mac, 20+ min)

```bash
docker buildx build --platform linux/amd64 -t ghcr.io/felores/thepopebot:event-handler-patched -f docker/event-handler/Dockerfile.patched --push .
```

### Deploy

```bash
# Server .env must have: EVENT_HANDLER_IMAGE_URL=ghcr.io/felores/thepopebot:event-handler-patched
ssh neo4j "cd /opt/thepopebot && docker compose up -d --force-recreate event-handler && docker restart coolify-proxy"
```

**Note**: Coolify's Traefik loses the route after container recreate — `docker restart coolify-proxy` is required.

## Patching JSX Components

The npm package ships **compiled `.js` files**, not `.jsx` source. The build step (`npm run build`) uses esbuild to compile `lib/chat/components/**/*.jsx` → `.js`. Next.js imports the `.js` output — patching `.jsx` files has **no effect**.

When overlaying a modified component in `Dockerfile.patched`:

```bash
# 1. Compile JSX to JS (from repo root)
npx esbuild lib/chat/components/chat-input.jsx \
  --bundle=false --format=esm --jsx=automatic \
  --outfile=docker/event-handler/chat-input.js

# 2. COPY the .js file in Dockerfile.patched (paths relative to repo root)
COPY docker/event-handler/chat-input.js ./node_modules/thepopebot/lib/chat/components/chat-input.js
```

Non-JSX files (e.g. `llm-providers.js`) can be copied directly without compilation.

## What Goes Where

| Change type | Branch | Example |
|-------------|--------|---------|
| Bug fix for everyone | `fix/*` off `main` | Fix chat input focus persistence |
| New feature for everyone | `feat/*` off `main` | Add new LLM models |
| App renaming / branding | `custom` | Rename to "DevBots" |
| Custom config / deployment | `custom` | Custom docker-compose, env tweaks |
| Docker patching infra | `custom` | Dockerfile.patched, compiled .js overlays |
