# COSMOS — What the Sky Hides

COSMOS is a browser-based, stylized 3D space explorer built with Svelte, Vite, and Three.js. It combines an interactive solar-system scene with selected deep-space destinations and an educational Orion constellation experiment.

## Features

- Interactive 3D solar-system scene with the Sun and eight planets.
- Stylized deep-space destinations: Proxima Centauri, Sirius, the Orion Nebula, and the Andromeda Galaxy.
- Object catalog with search and Solar / Deep Space filters.
- Click-to-select objects, drag-to-rotate, scroll-to-zoom, and smooth camera transitions.
- Pause/resume motion, scan logging, and a session-based discovery count.
- Science information with approximate distances and object facts.
- Orion Sky View vs. Space View experiment illustrating how a constellation's apparent pattern does not represent a flat physical arrangement.
- Responsive interface refinements for smaller screens.

## Technology

- Svelte 5
- Vite
- Three.js

## Run locally

Requirements: a current Node.js LTS release and npm.

```bash
npm install
npm run dev
```

Vite prints a local URL, typically `http://localhost:5173/`.

## Production build

```bash
npm run build
npm run preview
```

The production build is written to `dist/`.

## Deploy to Vercel

1. Push the project to the connected GitHub repository.
2. Import the repository into Vercel (or use the existing Vercel project).
3. For a Vite app, use build command `npm run build` and output directory `dist`.
4. Every push to the configured production branch should trigger a deployment.
5. Open the deployment URL and test the production version, not only localhost.

## Suggested demo flow

1. Enter the universe.
2. Select Earth, then another planet, to show object selection and camera movement.
3. Search for Orion Nebula or Andromeda Galaxy and open a deep-space destination.
4. Run a scan and show the discovery message.
5. Open the Orion Depth Experiment and switch between Sky View and Space View.
6. Explain why the constellation's depth view is an illustrative teaching model.

## Scientific limitations

This is an educational visualization, not a precision astronomy simulator. Planetary sizes and orbital spacing are stylized; deep-space positions are illustrative; and the Orion depth offsets are pedagogical rather than a precision 3D star catalogue. Object facts and distances are approximate reference information. The discovery log is stored in component state and resets when the page is reloaded.

## Before submission

- Run `npm run build`.
- Test selection, camera controls, search, filters, scans, pause/resume, and the Orion experiment.
- Open the deployed URL in a desktop and a narrow/mobile viewport.
- Check the browser console for runtime errors.
- Confirm that no `.env` files, credentials, or private tokens are committed.
