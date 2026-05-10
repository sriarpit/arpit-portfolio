# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Dev Commands

```bash
npm install        # first-time setup (requires .npmrc — see below)
npm run dev        # start dev server at http://localhost:5173 (--host exposes LAN)
npm run build      # tsc type-check + vite production build → dist/
npm run preview    # preview the production build locally
npm run lint       # eslint across all source files
```

No test suite exists in this project.

## GSAP Registry (.npmrc)

`@gsap/react` was previously locked to `npm.greensock.com`. A `.npmrc` file at the project root redirects it to the public registry:

```
@gsap:registry=https://registry.npmjs.org
```

This file is git-ignored. Re-create it if `npm install` fails with `ETIMEDOUT` on `npm.greensock.com`.

## Architecture Overview

### Render split: raw Three.js vs React Three Fiber

Two completely separate 3D rendering pipelines coexist:

- **Hero character** (`src/components/Character/`) — raw Three.js with a manually created `WebGLRenderer`, `Scene`, and `PerspectiveCamera` managed inside a `useEffect`. No R3F.
- **Tech Stack section** (`src/components/TechStack.tsx`) — React Three Fiber (`<Canvas>`) with Rapier physics (`@react-three/rapier`) and post-processing (`N8AO`). Lazy-loaded and desktop-only.

### Character loading pipeline

`public/models/character.enc` is an AES-CBC encrypted GLTF file. On load:
1. `decrypt.ts` fetches the `.enc` file, strips the 16-byte IV prefix, decrypts with key `"Character3D#@"` using Web Crypto API.
2. The decrypted buffer is turned into a blob URL and fed to `GLTFLoader` + `DRACOLoader` (decoder at `public/draco/`).
3. After load, `setCharTimeline()` and `setAllTimeline()` in `GsapScroll.ts` attach all GSAP ScrollTrigger timelines to the character and camera.

### GSAP scroll system

- `ScrollSmoother` (from `gsap-trial`) wraps `#smooth-wrapper > #smooth-content` and is initialised in `Navbar.tsx`. The smoother instance is exported as `smoother` for use across the app.
- All character scroll animations are in `src/components/utils/GsapScroll.ts` (`setCharTimeline` for landing/about/whatIDo, `setAllTimeline` for career).
- The Work section uses its own inline `ScrollTrigger` to pin and horizontally translate `.work-flex`.
- `gsap-trial/SplitText` powers the landing text reveal (`initialFX.ts`). Trial plugins cannot be used in production — replace with a licensed GSAP Club subscription for deployment.

### Loading / app boot sequence

`LoadingProvider` (`src/context/LoadingProvider.tsx`) gates the entire app:
1. Shows `<Loading>` overlay while `isLoading === true`.
2. `Scene.tsx` calls `setProgress()` (from `Loading.tsx`) to update the progress bar as the character model loads.
3. After the model resolves, `progress.loaded()` waits, then calls `light.turnOnLights()` and `animations.startIntro()` after a 2.5s delay.
4. `initialFX()` (called from `Loading.tsx`) unpauses `ScrollSmoother` and triggers the text animations.

### Responsive behaviour

`MainContainer.tsx` tracks `window.innerWidth > 1024` as `isDesktopView`:
- Desktop: `CharacterModel` renders outside the scroll wrapper (fixed position), `TechStack` is included.
- Mobile: `CharacterModel` renders inside `<Landing>`, `TechStack` is hidden entirely.

## Content Sections to Personalise

All placeholder content lives directly in the component files — there is no CMS or data layer:

| Section | File | What to update |
|---|---|---|
| Hero name / role | `src/components/Landing.tsx` | Name, role words (loop pair) |
| About bio | `src/components/About.tsx` | `<p>` body text |
| What I Do | `src/components/WhatIDo.tsx` | Skills tags, descriptions in both `.what-content` blocks |
| Career | `src/components/Career.tsx` | Role, company, year, description per `.career-info-box` |
| Work projects | `src/components/Work.tsx` | Project name, category, tools, `WorkImage` src per `.work-box` |
| Contact | `src/components/Contact.tsx` | Email, phone, social hrefs, copyright year |
| Navbar email | `src/components/Navbar.tsx` | `mailto:` href and display text |
| Social icons | `src/components/SocialIcons.tsx` | hrefs for GitHub, LinkedIn, Twitter, Instagram |
| Page title | `index.html` | `<title>` tag |
| Loading marquee | `src/components/Loading.tsx` | Scrolling role text |

## Tech stack logos (TechStack physics scene)

Sphere textures are loaded from `public/images/` as `.webp` files. The `imageUrls` array in `TechStack.tsx` maps directly to those files — add/remove entries there and drop the corresponding `.webp` into `public/images/`.
