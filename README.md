# Basic executable Svelte example

This repository contains a minimal Svelte + Vite setup you can run locally without additional scaffolding tools.

## Getting started
1. Install dependencies:
   ```bash
   npm install
   ```
2. Start the dev server:
   ```bash
   npm run dev -- --host
   ```
   Then open the printed URL (default `http://localhost:5173`).
3. Build for production:
   ```bash
   npm run build
   ```
4. Preview the production build:
   ```bash
   npm run preview -- --host
   ```

The example component renders a greeting, lets you update the displayed name, and includes a reactive click counter.
