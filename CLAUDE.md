# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenScreen is a free, open-source desktop screen recording and video editing application built with Electron, React, and TypeScript. It's an alternative to Screen Studio for creating product demos with cinematic pan/zoom effects, annotations, and professional export capabilities.

## Common Commands

```bash
# Development
npm run dev          # Start Vite dev server

# Building
npm run build        # Full production build (TypeScript + Vite + Electron Builder)
npm run build:mac    # macOS build only
npm run build:win    # Windows build only
npm run build:linux  # Linux build only

# Code Quality
npm run lint         # ESLint checking

# Testing
npm test             # Run Vitest tests once
npm run test:watch   # Watch mode for tests
```

## Architecture

### Multi-Window Electron App

The application uses a multi-window architecture where different UI modes are separate windows:

1. **HUD Overlay** (`windowType=hud-overlay`) - Floating recording controls (transparent, always-on-top)
2. **Source Selector** (`windowType=source-selector`) - Modal for selecting screen/app to record
3. **Editor** (`windowType=editor`) - Main video editing interface

The window type is determined by the URL query parameter in `src/App.tsx`, which routes to the appropriate component.

### Process Separation

- **Main Process** (`electron/main.ts`) - Window management, IPC handlers, system tray
- **Renderer Process** (`src/`) - React UI, video playback, editing logic
- **Preload Script** (`electron/preload.ts`) - Context bridge for secure IPC

### Key Directories

- `electron/` - Electron main process code (window lifecycle, IPC)
- `src/components/launch/` - Recording UI (LaunchWindow, SourceSelector)
- `src/components/video-editor/` - Main editor components (VideoEditor.tsx is the primary container)
- `src/components/video-editor/timeline/` - Timeline editing components
- `src/components/video-editor/videoPlayback/` - PixiJS rendering utilities
- `src/lib/exporter/` - Video export pipeline (MP4/GIF export, decoding, muxing)
- `src/hooks/` - React hooks including `useScreenRecorder.ts`

### Video Rendering Architecture

Video playback uses **PixiJS** with WebGL acceleration:

- `VideoPlayback.tsx` - Main PixiJS Application wrapper
- `videoPlayback/zoomTransform.ts` - Pan/zoom calculations
- `videoPlayback/layoutUtils.ts` - Video positioning
- `videoPlayback/focusUtils.ts` - Focus point clamping
- `videoPlayback/overlayUtils.ts` - UI indicators

### Export Pipeline

1. Decode video with `VideoDecoder` API (`videoDecoder.ts`)
2. Render frames through PixiJS (`frameRenderer.ts`)
3. Encode with `VideoEncoder` for MP4 or gif.js for GIF
4. Mux tracks with mp4box.js (`muxer.ts`)

## TypeScript Configuration

- Path alias: `@/*` maps to `src/*`
- Strict mode enabled
- No unused locals/parameters

## UI Components

Uses **shadcn/ui** (Radix UI primitives) with TailwindCSS:
- Components in `src/components/ui/`
- Theme configured in `components.json` (new-york style, lucide icons)
- Tailwind config uses "stone" base color with CSS variables for theming

## Testing

- Framework: Vitest (Node.js environment)
- Test files: `*.test.ts` or `*.spec.ts`
- Located in `src/lib/exporter/`

## Platform-Specific Notes

- **macOS**: Uses `hiddenInset` title bar style, traffic light positioning in top-left
- **Recordings**: Stored in `app.getPath('userData')/recordings`
- **Transparent windows**: Used for HUD overlay UI
- **System tray**: Integrated with icon swapping

## Important Dependencies

- `pixi.js` - 2D WebGL rendering for video playback
- `dnd-timeline` - Timeline component for video editing
- `mediabunny` - Video processing
- `mp4box` - MP4 container handling
- `gif.js` - GIF generation
- `gsap` - Animation library
- `motion` - React animation library
