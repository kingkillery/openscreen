# OpenScreen Demo Creator Skill

## Overview

OpenScreen is a free, open-source Electron + React + TypeScript application that lets users record their screen and produce polished product demos. It is the repository's core deliverable. This skill teaches agents how to understand, extend, and use the codebase to create demos autonomously.

**Tech stack:** Electron · React 18 · TypeScript · Vite · PixiJS · dnd-timeline · Biome (linter/formatter) · Vitest (unit tests) · Playwright (e2e tests)

---

## Repository Layout

```
openscreen/
├── electron/               # Electron main process
│   ├── main.ts             # App entry point, BrowserWindow setup
│   ├── preload.ts          # Context bridge (exposes IPC to renderer)
│   ├── windows.ts          # Window management helpers
│   └── ipc/handlers.ts     # All IPC handler registrations
├── src/                    # React renderer process
│   ├── components/
│   │   ├── launch/         # Launch window (source selector)
│   │   ├── ui/             # Reusable shadcn/Radix UI primitives
│   │   └── video-editor/   # Main editor (VideoEditor.tsx + sub-components)
│   │       ├── types.ts              # All shared TypeScript types
│   │       ├── projectPersistence.ts # Project serialization / validation
│   │       ├── timeline/             # Timeline panel components
│   │       └── videoPlayback/        # PixiJS-based playback engine
│   ├── hooks/              # Custom React hooks (recording, history, mic, etc.)
│   ├── contexts/           # React contexts (I18n, Shortcuts)
│   ├── lib/
│   │   ├── exporter/       # MP4 / GIF export pipeline
│   │   ├── recordingSession.ts  # ProjectMedia / RecordingSession types
│   │   ├── compositeLayout.ts   # Webcam composite layout logic
│   │   └── shortcuts.ts         # Keyboard shortcut definitions
│   └── utils/              # Aspect ratio, platform, time, test-id utils
├── tests/
│   ├── e2e/                # Playwright end-to-end tests
│   └── fixtures/           # Test fixture files
├── public/                 # Static assets (wallpapers, icons)
├── scripts/                # Build/maintenance scripts (e.g. i18n-check)
├── electron-builder.json5  # Electron packager config
├── vite.config.ts          # Vite + Electron plugin config
├── vitest.config.ts        # Vitest config
├── playwright.config.ts    # Playwright config
└── biome.json              # Linter/formatter config
```

---

## Key Concepts

### Project Data Model

A project (saved as JSON) follows `EditorProjectData` in `src/components/video-editor/projectPersistence.ts`:

```ts
interface EditorProjectData {
  version: number;          // PROJECT_VERSION = 2
  media: ProjectMedia;      // { screenVideoPath, webcamVideoPath? }
  editor: ProjectEditorState;
}

interface ProjectEditorState {
  wallpaper: string;        // Path like "/wallpapers/wallpaper3.jpg"
  shadowIntensity: number;  // 0–1
  showBlur: boolean;
  motionBlurAmount: number; // 0–1
  borderRadius: number;     // px
  padding: number;          // 0–100
  cropRegion: CropRegion;   // { x, y, width, height } in 0–1 range
  zoomRegions: ZoomRegion[];
  trimRegions: TrimRegion[];
  speedRegions: SpeedRegion[];
  annotationRegions: AnnotationRegion[];
  aspectRatio: AspectRatio; // "16:9" | "9:16" | "1:1" | "4:3" | "4:5" | "16:10" | "10:16" | "native"
  webcamLayoutPreset: "picture-in-picture" | "vertical-stack";
  webcamPosition: { cx: number; cy: number } | null;
  exportQuality: "medium" | "good" | "source";
  exportFormat: "mp4" | "gif";
  // GIF-specific
  gifFrameRate: 15 | 20 | 25 | 30;
  gifLoop: boolean;
  gifSizePreset: "medium" | "large" | "original";
}
```

### ZoomRegion

```ts
interface ZoomRegion {
  id: string;       // e.g. "zoom-1"
  startMs: number;  // milliseconds from video start
  endMs: number;
  depth: 1 | 2 | 3 | 4 | 5 | 6;  // 1=1.25× … 6=5.0× scale
  focus: { cx: number; cy: number }; // normalized (0–1)
}
```

Depth → scale mapping (from `ZOOM_DEPTH_SCALES`):
| depth | scale |
|-------|-------|
| 1     | 1.25× |
| 2     | 1.50× |
| 3     | 1.80× |
| 4     | 2.20× |
| 5     | 3.50× |
| 6     | 5.00× |

### AnnotationRegion

```ts
interface AnnotationRegion {
  id: string;
  startMs: number;
  endMs: number;
  type: "text" | "image" | "figure";
  content: string;         // legacy / current text
  textContent?: string;
  imageContent?: string;   // data URL for image annotations
  position: { x: number; y: number }; // percentage (0–100)
  size: { width: number; height: number }; // percentage
  style: AnnotationTextStyle;
  zIndex: number;
  figureData?: FigureData; // for "figure" type (arrow shapes)
}
```

### TrimRegion / SpeedRegion

Both share the same shape `{ id, startMs, endMs }` plus `speed` for `SpeedRegion`. Trim regions mark segments to cut out; speed regions mark segments to play at a different rate (0.25×–2×).

---

## Development Workflow

### Install Dependencies
```bash
npm install
```

### Run in Development Mode (Electron)
```bash
npm run dev
```

### Lint / Format
```bash
npm run lint          # Biome check
npm run lint:fix      # Biome auto-fix
npm run format        # Biome format
```

### Run Unit Tests
```bash
npm test              # Vitest (all unit tests, single run)
npm run test:watch    # Vitest in watch mode
```

### Run E2E Tests
```bash
npm run test:e2e      # Playwright
```

### Build (packages Electron app)
```bash
npm run build         # all platforms
npm run build:mac
npm run build:win
npm run build:linux
```

---

## How to Create a Demo Programmatically

Agents can generate a valid OpenScreen project JSON to script the entire editing pipeline without touching the UI.

### Step 1 – Prepare source video

The video must be accessible as a local file path. Provide it as `screenVideoPath` inside `media`. An optional `webcamVideoPath` can also be included.

### Step 2 – Build `ProjectEditorState`

Use the helpers in `projectPersistence.ts` to construct and validate the state:

```ts
import { normalizeProjectEditor, createProjectData } from
  'src/components/video-editor/projectPersistence';
import type { ProjectMedia } from 'src/lib/recordingSession';

const media: ProjectMedia = { screenVideoPath: '/absolute/path/to/recording.mp4' };

const editor = normalizeProjectEditor({
  wallpaper: '/wallpapers/wallpaper3.jpg',
  padding: 60,
  borderRadius: 16,
  shadowIntensity: 0.6,
  motionBlurAmount: 0.35,
  aspectRatio: '16:9',
  exportQuality: 'good',
  exportFormat: 'mp4',

  // Zoom into the top-left quadrant at 3 seconds for 2 seconds
  zoomRegions: [{
    id: 'zoom-1',
    startMs: 3000,
    endMs: 5000,
    depth: 3,          // 1.8× zoom
    focus: { cx: 0.25, cy: 0.25 },
  }],

  // Add a text label visible for the first 4 seconds
  annotationRegions: [{
    id: 'ann-1',
    startMs: 0,
    endMs: 4000,
    type: 'text',
    content: 'Hello, World!',
    position: { x: 50, y: 10 },
    size: { width: 40, height: 10 },
    style: {
      color: '#ffffff',
      backgroundColor: 'transparent',
      fontSize: 32,
      fontFamily: 'Inter',
      fontWeight: 'bold',
      fontStyle: 'normal',
      textDecoration: 'none',
      textAlign: 'center',
    },
    zIndex: 1,
  }],

  // Trim the last 2 seconds (assuming 10s total video)
  trimRegions: [{
    id: 'trim-1',
    startMs: 8000,
    endMs: 10000,
  }],
});

const project = createProjectData(media, editor);
// Serialize: JSON.stringify(project)
```

### Step 3 – Save the project file

The project JSON can be written to disk (e.g. `~/Documents/my-demo.openscreen`) and then opened from the OpenScreen application via **File → Open Project** or the "Open existing project" button in the launch window.

The file extension used by the app is `.openscreen`.

### Step 4 – Export from the UI (or programmatically via IPC)

The exporter lives in `src/lib/exporter/`. For Electron IPC-driven export without a UI, call the relevant IPC channels exposed in `electron/ipc/handlers.ts` and `electron/preload.ts`.

---

## Adding a New Feature for Demo Creation

When implementing a new editing feature (e.g. a new annotation type, a new transition, a new export format):

1. **Add the type** to `src/components/video-editor/types.ts`.
2. **Update persistence** in `src/components/video-editor/projectPersistence.ts` — add normalization logic and default values.
3. **Add UI controls** in the appropriate panel inside `src/components/video-editor/`.
4. **Hook into the playback engine** via `src/components/video-editor/videoPlayback/` (PixiJS).
5. **Hook into the exporter** via `src/lib/exporter/` if the feature affects the exported video.
6. **Write unit tests** alongside the changed file (Vitest; see existing `*.test.ts` files for patterns).
7. **Run linter** with `npm run lint:fix` before committing.

---

## Testing Conventions

- Unit tests live next to the file they test as `<filename>.test.ts`.
- Use `vitest` APIs (`describe`, `it`, `expect`).
- Property-based tests use `fast-check` (already a dev-dependency).
- E2E tests live in `tests/e2e/` and use Playwright.
- Test IDs are generated by `src/utils/getTestId.ts` — use `data-testid` attributes for Playwright selectors.

---

## Wallpapers

18 built-in wallpapers are available as `/wallpapers/wallpaper1.jpg` through `/wallpapers/wallpaper18.jpg`. The `wallpaper` field also accepts:

- A CSS solid colour string (e.g. `"#1a1a2e"`).
- A CSS gradient string (e.g. `"linear-gradient(135deg, #667eea 0%, #764ba2 100%)"`).
- A `file://` URL pointing to a custom image on disk.

---

## Aspect Ratios

Available values for `aspectRatio` (from `src/utils/aspectRatioUtils.ts`):

| Value    | Use case                  |
|----------|---------------------------|
| `16:9`   | Standard widescreen       |
| `9:16`   | Mobile / vertical video   |
| `1:1`    | Square (social media)     |
| `4:3`    | Classic screen            |
| `4:5`    | Instagram portrait        |
| `16:10`  | MacBook / wide            |
| `10:16`  | Tall portrait             |
| `native` | Match source video ratio  |

---

## Export Quality Reference

| `exportQuality` | Description                    |
|-----------------|--------------------------------|
| `"medium"`      | Compressed, smaller file size  |
| `"good"`        | Balanced (default)             |
| `"source"`      | Lossless / maximum quality     |

---

## Common Pitfalls

- `startMs` must be strictly less than `endMs`; the normalizer enforces a minimum gap of 1 ms.
- `focus.cx` and `focus.cy` must be in `[0, 1]`.
- `position.x/y` for annotations are percentages `[0, 100]`.
- When generating IDs, use a consistent prefix + counter (e.g. `"zoom-1"`, `"zoom-2"`) or a UUID from the `uuid` package.
- The `content` field of an `AnnotationRegion` is legacy but still read; always also set `textContent` for text annotations.
- Project version must be `2` (the value of `PROJECT_VERSION`).

---

## Quick-Start Checklist for Autonomous Demo Creation

- [ ] Identify the source screen recording file path.
- [ ] Decide the background: wallpaper index, solid colour, or gradient.
- [ ] Plan zoom regions: for each key moment, define `startMs`, `endMs`, `depth`, and `focus`.
- [ ] Plan annotation regions: text labels, arrows, or image overlays.
- [ ] Plan trim regions: any sections to remove.
- [ ] Optionally plan speed regions to speed up/slow down segments.
- [ ] Choose `aspectRatio` and `exportQuality`.
- [ ] Call `normalizeProjectEditor(...)` and `createProjectData(...)` to produce the JSON.
- [ ] Write the JSON to a `.openscreen` file and open it in the app.
- [ ] Export the final video from the editor.
