# Self-Hosted Agentic Instructional Video Studio — Build and Design Spec

## 1. Product vision

Build a self-hosted web application that helps a user create high-quality instructional videos, playlists, and supporting learning materials from topics they personally want to learn.

The product is not merely a “generate a video” tool. It is a **filesystem-first agentic learning studio** where a user can create a curriculum, generate a learning plan, produce multiple instructional videos, review and revise scene-level chunks, choose a consistent voice, generate thumbnails, and optionally publish the finished work to YouTube.

The application should feel like a local creative workbench:

- The user creates a curriculum or course.
- The app proposes a structured table of contents.
- The agent generates content orders, lessons, scenes, scripts, visuals, narration, and renders.
- The user can watch previews, click or select chunks, and give text or microphone feedback.
- The agent revises the smallest reasonable unit.
- The app keeps artifacts organized on disk.
- The user can publish to YouTube after explicit approval.

The system should optimize for **high signal-to-noise instructional quality**, not bulk content generation.

---

## 2. Guiding principles

### 2.1 Prefer boring, inspectable architecture

Use simple technology until complexity is clearly justified.

Prefer:

- Static Vite frontend.
- Python backend.
- FastAPI.
- Pydantic AI.
- Filesystem volume as source of truth.
- SQLite as a rebuildable metadata and job index.
- Subprocess tools with clear manifests.
- ADRs for important decisions.

Avoid initially:

- Distributed job queues.
- Microservices.
- Postgres.
- Object storage.
- Kubernetes.
- Complex plugin marketplaces.
- Durable workflow engines.
- Overly elaborate multi-agent systems.

Complexity is allowed when it directly supports the core product experience.

### 2.2 Files are the source of truth

The curriculum workspace should be a folder on disk. The user should be able to inspect it, back it up, zip it, move it, or edit some files manually.

SQLite may be used as a useful local index, but important project artifacts should exist as files.

The database is the map. The filesystem is the territory.

### 2.3 Agents should apply structured patches, not mutate invisibly

The agent should be able to propose or apply changes to specific artifacts:

- Curriculum plan.
- Content order.
- Lesson brief.
- Scene script.
- Visual specification.
- Manim code.
- Voiceover.
- Rendered video chunk.
- Thumbnail.
- YouTube metadata.

Every generated or modified artifact should be traceable to a generation run.

### 2.4 Regenerate the smallest reasonable unit

The user should not have to regenerate a whole video because one 20-second section is weak.

The basic loop should be:

```text
select artifact or timeline range
→ give typed or spoken feedback
→ agent proposes a structured edit
→ regenerate the smallest affected unit
→ preview
→ accept or reject
```

### 2.5 Human approval before serious actions

The agent may freely draft, propose, render local previews, and organize local artifacts.

The agent must require explicit approval before:

- Publishing or making videos public.
- Uploading to YouTube.
- Deleting accepted/final artifacts.
- Running destructive filesystem actions.
- Using external paid APIs beyond configured auto-run limits.
- Sending private video/source material to external review tools.
- Changing OAuth credentials or secrets.

### 2.6 ADRs are mandatory

The coding agent must maintain Architecture Decision Records in:

```text
docs/ADRs/
```

ADRs are required for changes involving:

- Storage model.
- Job execution model.
- Tool execution/security model.
- Agent framework.
- Renderer architecture.
- YouTube/OAuth integration.
- Secrets management.
- Major frontend architecture.
- API contract changes.
- Database schema philosophy.
- Import/export behavior.

---

## 3. Confirmed technology choices

### 3.1 Frontend

Use:

```text
Vite
React
TypeScript
Tailwind CSS
shadcn/ui where helpful
```

The frontend should be a static Vite app. It should not own the backend or media pipeline.

The UI should feel like a modern creative/editorial workbench, not a generic admin dashboard.

### 3.2 Backend

Use:

```text
Python
FastAPI
Pydantic
Pydantic AI
SQLite
Filesystem volume
```

FastAPI is the main API server.

The Pydantic AI agent runner should run **in the same Python backend process for the first complete version**.

Do not introduce a separate worker service unless the implementation hits concrete pain.

### 3.3 Agent framework

Use **Pydantic AI** as the primary agent framework.

Reasons:

- Strong Python typing.
- Pydantic schemas.
- Typed tools.
- Structured outputs.
- Fits the user’s existing familiarity.
- Works naturally with Python-native media and rendering tools.

### 3.4 Durable execution

Do **not** include DBOS, Temporal, Prefect, or Restate in the first complete version.

Pydantic AI does support durable execution integrations such as DBOS, and DBOS can checkpoint durable workflows using Postgres or SQLite, but this is not required for the first version. Durable workflows should remain a future extension point rather than a first-version dependency.

### 3.5 Rendering

Initial renderer set:

```text
Manim
FFmpeg
```

Design the architecture so additional renderers can be added later, especially:

```text
HTML/Markdown scene renderer
HyperFrames-style renderer
Chart renderer
Diagram renderer
Avatar/presenter renderer
Image generation renderer
```

### 3.6 Speech-to-text

Support microphone feedback in the first complete version.

Use an external transcription provider initially, with **GPT-4o mini Transcribe** as the likely first provider. OpenAI documents GPT-4o mini Transcribe as a speech-to-text model.

Local transcription can be a later provider.

### 3.7 Text-to-speech

Support **Qwen3-TTS** as the first-class/default TTS provider.

Qwen3-TTS is an open-source TTS model family that supports speech generation, voice design, voice cloning, and local inference patterns.

The app should still define a generic TTS provider interface so another provider can be swapped in later.

### 3.8 Image generation for thumbnails

Support an image generation provider interface.

Prefer GPT Image 2 as the first external image-generation provider if available/configured. OpenAI currently documents GPT Image 2 as its state-of-the-art image generation model for API use.

Do not hardcode the whole app around a single image model. Use a provider abstraction.

### 3.9 YouTube integration

Support YouTube OAuth and publishing in the first complete version.

YouTube Data API supports external app integrations with resources like videos and playlists, and modifying user/channel data requires OAuth authorization.

YouTube publishing should include:

- Connect YouTube account.
- Display connected channel.
- Create playlist.
- Upload video.
- Add video to playlist.
- Set title.
- Set description.
- Set visibility.
- Upload thumbnail.
- Add chapters in description.
- Captions later or optional.
- Auto-translation later.

Default visibility should be **private**.

The user should be able to later flip selected videos or an entire playlist from private to public.

---

## 4. High-level architecture

```text
Browser
  ↓
Static Vite React frontend
  ↓
FastAPI backend
  ↓
Pydantic AI coordinator agent
  ↓
SQLite metadata/job index
  ↓
Filesystem curriculum workspaces
  ↓
Tool runner / subprocess tools
  ↓
Manim, FFmpeg, Qwen3-TTS, transcription, image generation, YouTube API
```

### 4.1 Runtime shape

The first complete version should run as one backend process plus one frontend dev server in development.

Development experience should feel simple:

```text
start frontend
start backend
open browser
```

Possible commands:

```bash
pnpm dev
uv run app
```

or via a single convenience script:

```bash
pnpm dev:all
```

### 4.2 No separate Python worker initially

The backend should include:

- API routes.
- Agent runner.
- Job runner.
- Tool runner.
- SQLite access.
- Filesystem artifact management.

Long-running jobs should not block request handlers. They should be launched as background jobs tracked in SQLite.

However, the job runner may live in the same Python process for now.

---

## 5. Repository structure

Suggested initial structure:

```text
repo/
  apps/
    web/
      package.json
      vite.config.ts
      src/
        app/
        components/
        features/
        lib/
        routes/
        styles/
    api/
      pyproject.toml
      src/
        app/
          main.py
          api/
          agents/
          artifacts/
          jobs/
          tools/
          renderers/
          youtube/
          tts/
          transcription/
          thumbnails/
          storage/
          settings/
          schemas/
          utils/
      tests/
  docs/
    ADRs/
    product/
    development/
  examples/
    curricula/
    tools/
  scripts/
  docker/
  README.md
```

The exact layout may be adjusted by the coding agent, but the separation between frontend, backend, docs, and examples should remain clear.

---

## 6. Storage model

### 6.1 Canonical storage

Use a mounted data volume, for example:

```text
/data
  /curricula
  /tools
  /global
  /cache
  /exports
  app.sqlite
```

Each curriculum should be a portable folder.

Example:

```text
/data/curricula/nat-from-first-principles/
  curriculum.yaml
  creator_notes.md
  style_kit.yaml
  voice_profile.yaml
  publishing.yaml

  /orders
  /lessons
  /scenes
  /assets
  /renders
  /sources
  /reviews
  /runs
  /exports
  /youtube
```

### 6.2 SQLite role

Use SQLite for:

- Curriculum index.
- Artifact index.
- Job tracking.
- Tool call logs.
- Generation run index.
- Search metadata.
- UI state where appropriate.
- OAuth connection metadata, but not necessarily raw secrets unless handled carefully.

SQLite should be treated as rebuildable wherever possible.

If SQLite becomes corrupted or stale, the app should eventually support:

```text
Admin → Rebuild index from filesystem
```

### 6.3 SQLite hardening

The backend should configure SQLite carefully:

- Use WAL mode.
- Enable foreign keys.
- Use migrations from day one.
- Use transactions around related writes.
- Avoid writing large binary blobs into SQLite.
- Store large media artifacts as files.
- Keep sidecar metadata files next to important artifacts.
- Provide periodic integrity checks if practical.
- Provide export/backup tools.

Suggested tooling:

```text
SQLAlchemy 2.x
Alembic migrations
Pydantic schemas for API/data validation
```

SQLModel is acceptable if the coding agent strongly prefers it, but the spec should not depend on it.

### 6.4 Artifact sidecars

Every important artifact should have metadata either in SQLite, a sidecar file, or both.

Example:

```text
/assets/manim/nat-table/v1/
  main.py
  preview.mp4
  metadata.yaml
```

Metadata example:

```yaml
id: asset_manim_nat_table_v1
kind: manim_scene
created_at: "2026-08-13T23:03:00-04:00"
created_by_run: run_2026_08_13_230300_001
status: accepted
parent: null
used_by:
  - scene_004
notes: "Initial NAT table animation."
```

---

## 7. Core domain objects

The implementation should define these concepts clearly, but the exact schema may be refined during build.

### 7.1 Curriculum

A curriculum is the top-level workspace.

Examples:

- Linear Algebra.
- PyTorch From First Principles.
- NAT and Home Networking.
- Kubernetes Internals.

Contains:

- Title.
- Slug.
- Description.
- Audience.
- Desired depth.
- Creator notes.
- Voice profile.
- Style kit.
- Tool policy.
- Publishing settings.
- Playlists.
- Content orders.
- Lessons.
- Artifacts.

### 7.2 Playlist

A playlist is an ordered learning path inside a curriculum.

Contains:

- Title.
- Description.
- Ordered lesson/video list.
- YouTube playlist mapping if published.
- Visibility default.
- Publishing status.

### 7.3 Content order

A content order is an actionable request for one piece or bundle of content.

Examples:

- “Explain what problem NAT solves.”
- “Create a 45-second Short on NAT tables.”
- “Make a thumbnail for Lesson 3.”
- “Generate the table of contents for the course.”

Contains:

- Format.
- Audience.
- Goal.
- Constraints.
- Status.
- Related lesson/scene/artifacts.
- Review state.

### 7.4 Lesson

A lesson is a structured instructional unit, usually corresponding to one long-form video.

Contains:

- Learning objectives.
- Prerequisites.
- Outline.
- Script.
- Scenes.
- Claims.
- Sources.
- Review results.
- Render status.
- YouTube metadata.

### 7.5 Scene

A scene is the smallest major renderable/editable chunk.

Contains:

- Scene title.
- Teaching goal.
- Narration text.
- Visual spec.
- Renderer type.
- Source files.
- Audio.
- Captions/alignment if available.
- Preview render.
- Review notes.
- Status.

### 7.6 Asset

An asset is any reusable media or source artifact.

Examples:

- Manim animation.
- Audio narration.
- Generated thumbnail.
- Web image.
- Chart.
- Diagram.
- Rendered scene chunk.
- Final MP4.
- Captions file.
- Source snapshot.

### 7.7 Generation run

A generation run records what the agent did.

Contains:

- Run ID.
- User request.
- Agent/system prompt version.
- Model used.
- Tool calls.
- Inputs.
- Outputs.
- Files changed.
- Artifacts created.
- Review results.
- Errors.
- Parent run if applicable.

### 7.8 Review

A review records quality feedback.

Review types:

- Claim review.
- Pedagogy review.
- Visual review.
- Pacing review.
- Sound/voice review.
- Publish readiness review.

### 7.9 Tool call

A tool call records invocation of a capability.

Contains:

- Tool ID.
- Input.
- Output.
- Artifacts created.
- Runtime.
- Exit code.
- Logs.
- Permissions used.
- Error state.

---

## 8. Agent model

### 8.1 User-facing model

The user should experience **one central coordinator agent per curriculum**.

The user can talk to the curriculum agent and ask it to:

- Create a course.
- Make a plan.
- Generate lessons.
- Render videos.
- Review a scene.
- Revise a weak section.
- Prepare YouTube metadata.
- Publish after approval.
- Check project status.

The app should not expose a confusing “company of agents” UI.

### 8.2 Internal agent organization

Internally, the coordinator may invoke specialist modes or subagents.

Possible specialists:

```text
Curriculum planner
Researcher
Lesson writer
Visual planner
Scene renderer coordinator
Voice/narration coordinator
Reviewer
Thumbnail generator
YouTube publisher
```

These may be implemented as:

- Separate Pydantic AI agents.
- Different system prompts.
- Structured workflows.
- Typed tools.
- A combination.

The exact implementation may be decided during development, but the UI should remain centered around one primary conversational agent.

### 8.3 Agent scope

Conversations should be scope-aware.

Possible scopes:

```text
global
curriculum
playlist
lesson
scene
artifact
timeline range
YouTube publishing package
```

When the user selects a scene or timeline chunk and gives feedback, the selected scope should be included in the agent context.

### 8.4 Agent responsibilities

The coordinator agent should be able to:

- Load relevant curriculum context.
- Inspect artifacts.
- Search registered sources.
- Read creator notes.
- Use enabled tools.
- Propose plans.
- Apply approved patches.
- Create jobs.
- Report status.
- Explain what changed.
- Ask for human approval when required.

### 8.5 Agent non-goals

The agent should not:

- Silently publish to YouTube.
- Silently delete accepted/final artifacts.
- See raw secrets if avoidable.
- Run arbitrary shell commands without tool permission.
- Modify architecture without an ADR.
- Treat unreviewed content as publish-ready.

---

## 9. Tool system

### 9.1 Tool philosophy

The tool system should make new functionality easy to add over time without requiring major backend rewrites.

A tool can wrap:

- Python function.
- Pydantic AI tool.
- Command-line executable.
- `uvx` command.
- Docker container.
- Local API endpoint.
- External API.
- Future MCP server.

### 9.2 First-version decision

For the first complete version:

```text
Pydantic AI tools are the agent-facing interface.
Backend services implement tool behavior.
Manifest-defined external tools may be supported if practical.
MCP is optional later, not required for first version.
```

### 9.3 Tool contract

Each tool should have:

- ID.
- Name.
- Description.
- Input schema.
- Output schema.
- Permissions.
- Timeout.
- Required environment variables.
- Artifact output expectations.
- Error behavior.
- Logging behavior.

### 9.4 Tool permissions

Permission categories:

```text
safe_local
expensive_local
networked
external_api
destructive
publishing
```

Examples:

- Rendering a local Manim preview: safe or expensive local.
- Calling GPT Image 2: external API.
- Uploading to YouTube: publishing.
- Deleting rejected render cache: destructive.
- Deleting final accepted video: destructive and requires approval.

### 9.5 Tool output

Tools should return structured output.

Bad:

```json
{"status": "done"}
```

Good:

```json
{
  "status": "success",
  "summary": "Rendered scene 004 preview.",
  "artifacts": [
    {
      "id": "render_scene_004_preview_v2",
      "type": "video",
      "path": "renders/drafts/scene_004_preview_v2.mp4",
      "duration_seconds": 31.8
    }
  ],
  "warnings": [],
  "logs_path": "runs/run_123/logs.txt"
}
```

### 9.6 Built-in tools

First built-in tools should include:

```text
Artifact search/read/write
Curriculum planning
Lesson planning
Scene planning
Manim render
FFmpeg compose
Qwen3-TTS synthesize
Speech-to-text transcription
Thumbnail generation
YouTube OAuth/publish
Review/quality gate checker
```

---

## 10. Job execution

### 10.1 First-version decision

Use a **SQLite-backed job queue** running in the same FastAPI backend process.

Do not introduce Redis, Celery, DBOS, or Temporal initially.

### 10.2 Job types

Expected jobs:

- Generate curriculum plan.
- Generate lesson outline.
- Generate scenes.
- Render Manim scene.
- Synthesize voiceover.
- Compose video with FFmpeg.
- Generate thumbnail.
- Run review.
- Transcribe microphone feedback.
- Upload to YouTube.
- Create/update YouTube playlist.
- Export curriculum zip.
- Rebuild SQLite index.

### 10.3 Job status

Statuses:

```text
queued
running
waiting_for_approval
succeeded
failed
cancelled
```

Jobs should expose:

- Progress summary.
- Logs.
- Current step.
- Artifacts created.
- Error messages.
- Retry action if appropriate.

### 10.4 Job events

The frontend should be able to show live-ish updates.

Implement with one of:

- Server-sent events.
- WebSocket.
- Polling for first version if simpler.

Favor simple implementation first.

---

## 11. Rendering pipeline

### 11.1 Scene as standard unit

Every renderable scene should eventually produce:

```text
video chunk
duration
dimensions
audio track or reference
caption/alignment metadata if available
source files
logs
preview thumbnail
```

This standard output should be the same whether the scene came from Manim, HTML, Markdown, charts, images, or future avatar tools.

### 11.2 Initial renderers

First complete version:

```text
Manim renderer
FFmpeg compositor
Qwen3-TTS narration
```

### 11.3 Future renderers

The architecture should allow:

```text
HTML/Markdown scene renderer
HyperFrames-style renderer
Chart renderer
Diagram renderer
Code/terminal renderer
Image scene renderer
Avatar teacher renderer
```

### 11.4 FFmpeg composition

FFmpeg should assemble:

- Scene videos.
- Narration audio.
- Optional sound effects.
- Captions if available.
- Intro/outro if configured.
- Final MP4.
- Short clips if generated.

### 11.5 Chunked video generation

Videos should be generated in separate chunks/scenes.

The editor should allow:

- Preview one scene.
- Regenerate one scene.
- Regenerate narration only.
- Regenerate visual only.
- Recompose full video after accepted scene changes.

---

## 12. Voice and TTS

### 12.1 Voice consistency

The app must support consistent voice selection per curriculum or playlist.

Define a `VoiceProfile`.

Possible fields:

```yaml
id: voice_profile_default
provider: qwen3_tts
voice_id: selected_or_generated_voice_id
reference_audio_path: null
reference_text: null
style_prompt: "Clear, calm, high-signal technical instructor."
speed: 1.0
pitch: default
pronunciation_dictionary_path: voice/pronunciations.yaml
```

### 12.2 Qwen3-TTS implementation notes

Qwen3-TTS may support different voice modes, including predefined speakers, voice design, or reference-audio voice cloning depending on the chosen model/configuration. The implementation should hide this behind a provider interface.

The app should allow the user to:

- Pick a voice.
- Save it to the curriculum.
- Preview sample narration.
- Reuse the same voice across videos.
- Maintain a pronunciation dictionary.

### 12.3 Pronunciation dictionary

Support domain-specific pronunciations.

Examples:

```yaml
kubectl: "cube control"
etcd: "et-see-dee"
CIDR: "sigh-der"
Qwen: "kwen"
Manim: "man-im"
PyTorch: "pie torch"
```

Exact pronunciation format may depend on provider support. If provider-specific phoneme support is not available, use pronunciation hints in prompt/instructions.

---

## 13. Microphone feedback

### 13.1 Core interaction

The user should be able to select a scene, timestamp range, or artifact and click a microphone button.

Example user feedback:

```text
This part is too abstract. Slow it down and show the NAT table filling one row at a time.
```

The app should:

1. Record audio.
2. Transcribe it.
3. Attach the transcript to the selected scope.
4. Ask the agent to parse it into structured edit intent.
5. Show or apply a proposed revision.
6. Create a job to regenerate the affected chunk.

### 13.2 Transcription provider

First provider:

```text
OpenAI GPT-4o mini Transcribe
```

Use local transcription later if desired.

### 13.3 Technical vocabulary context

When transcribing, include optional context or glossary where supported.

The app should pass terms related to the current curriculum, such as:

- Kubernetes.
- NAT.
- PyTorch.
- Qwen3-TTS.
- Manim.
- FFmpeg.
- CIDR.
- etcd.
- kubectl.

---

## 14. Image and thumbnail generation

### 14.1 Provider abstraction

Define:

```text
ImageGenerationProvider
  - generate_image(prompt, size, output_path, style_context)
  - edit_image(input_image, prompt, output_path)
  - describe_or_review_image(image_path), optional
```

First likely provider:

```text
OpenAI GPT Image 2
```

Allow future providers:

```text
local image model
Nano Banana-style provider
custom command tool
manual upload
```

### 14.2 Thumbnail workflow

For each video, the app should support:

- Generate thumbnail concepts.
- Generate thumbnail image.
- Allow manual upload/replace.
- Store thumbnail artifact.
- Associate thumbnail with YouTube metadata.
- Require approval before upload.

Thumbnail generation should consider:

- Curriculum style kit.
- Video title.
- Main concept.
- Visual simplicity.
- Readability at small sizes.
- Consistency across playlist.

---

## 15. YouTube integration

### 15.1 OAuth flow

Implement backend-managed OAuth.

Flow:

```text
Frontend "Connect YouTube"
  → FastAPI starts OAuth
  → User grants permissions in browser
  → FastAPI receives callback
  → Backend stores tokens securely
  → App shows connected channel
  → Agent can use YouTube tools without seeing raw secrets
```

Do not give raw YouTube credentials to the coding agent or runtime agent when avoidable.

### 15.2 Scopes

Use the narrowest OAuth scopes practical for the supported features.

The app needs capabilities for:

- Video upload.
- Video metadata update.
- Playlist creation/update.
- Add video to playlist.
- Thumbnail upload.
- Captions later/optional.

YouTube’s API documents videos, playlists, thumbnails, captions, and OAuth-authenticated modification operations.

### 15.3 Token storage

Use safe local storage.

Recommended first version:

- Store OAuth token metadata in SQLite.
- Store sensitive token material in a local secrets file with restrictive filesystem permissions, or in an encrypted local store if practical.
- Never expose raw token values to the agent prompt.
- Backend tools use tokens internally.

If encryption is not implemented in first version, the app must make local security assumptions explicit and keep file permissions restrictive.

### 15.4 Publishing package

Each publishable video should have a publishing package:

```yaml
video_artifact_id: final_lesson_001_mp4
title: "How NAT Actually Knows Where Replies Go"
description_path: youtube/lesson_001_description.md
thumbnail_artifact_id: thumbnail_lesson_001_v2
visibility: private
playlist_id: youtube_playlist_abc123
chapters:
  - time: "00:00"
    title: "The problem NAT solves"
  - time: "01:42"
    title: "Private vs public IPs"
  - time: "03:15"
    title: "The NAT table"
captions_path: null
status: ready_for_review
```

### 15.5 Visibility

Default visibility:

```text
private
```

The user should be able to:

- Upload individual videos as private.
- Change a video to unlisted/public.
- Flip an entire playlist public later.
- Keep drafts local only.

### 15.6 Upload approval

Publishing must require explicit human approval.

The UI should show:

- Final video preview.
- Title.
- Description.
- Thumbnail.
- Visibility.
- Playlist.
- Chapters.
- Quality gate status.
- Confirmation button.

No automatic public publishing.

### 15.7 Captions

Captions may be optional/later, but design for them.

Future support:

- Generate captions.
- Upload captions.
- Auto-translate captions.
- Download/update captions.

---

## 16. Quality gates

### 16.1 Publish readiness

Before a video can be uploaded, verify:

```text
all scenes rendered
final MP4 exists
voiceover completed
no failed required render jobs
thumbnail exists or user skipped thumbnail
title exists
description exists
visibility chosen
playlist chosen or skipped
chapters valid if present
human approval granted
```

### 16.2 Review blockers

The review system should support blockers.

Examples:

- Unsupported factual claim.
- Render failed.
- Missing audio.
- Empty scene.
- Description missing.
- YouTube metadata incomplete.
- Thumbnail generation failed.
- External API error.

### 16.3 Non-blocking warnings

Examples:

- Pacing may be too fast.
- Scene has high text density.
- No captions generated.
- Thumbnail may be too detailed.
- Lesson lacks exercises.
- Sources are old.

### 16.4 Review types

First version should include at least a simple structured publish-readiness review.

Future or stretch:

- Claim-level factual review.
- Pedagogy review.
- Visual clarity review.
- Voice/pacing review.
- Whole-video review with Gemini-style video understanding.
- Accessibility review.

---

## 17. Frontend product experience

### 17.1 Design goal

The UI should feel like an approachable creative studio for generating and revising instructional videos.

It should not feel like:

- A generic CRUD admin panel.
- A complicated NLE clone.
- A chatbot with some files bolted on.
- A dashboard full of inscrutable agent logs.

It should feel like:

```text
curriculum planner
+ video production board
+ scene editor
+ agent chat
+ artifact browser
+ render queue
+ publish checklist
```

### 17.2 Main pages

Suggested pages:

```text
Dashboard
Curriculum detail
Playlist/table-of-contents view
Content orders view
Lesson/scene editor
Video preview/editor
Tool registry/settings
Voice settings
YouTube publishing settings
Render/job queue
Admin/storage view
```

### 17.3 Dashboard

Show:

- Curricula.
- Recent work.
- Render jobs.
- Drafts needing review.
- YouTube publishing status.
- Disk usage summary.

### 17.4 Curriculum view

Show:

- Curriculum title and goal.
- Audience/depth.
- Table of contents.
- Playlists.
- Content orders.
- Agent conversation.
- Create lesson/video actions.

### 17.5 Playlist/table-of-contents view

This should be a primary view.

Show:

- Ordered lessons/videos.
- Status per video.
- Draft/render/publish state.
- Visibility.
- YouTube mapping if uploaded.
- Ability to reorder.
- Ability to ask agent to revise structure.

### 17.6 Lesson/scene editor

Show:

- Lesson outline.
- Scene list.
- Script/narration.
- Visual spec.
- Claims/review notes if available.
- Render status.
- Scene preview.
- Agent chat scoped to lesson or scene.

### 17.7 Video preview/editor

First version should support:

- Scene timeline.
- Chunk preview.
- Transcript/narration view.
- Select a scene or chunk.
- Type feedback.
- Record microphone feedback.
- Regenerate selected chunk.
- Compare previous and revised chunk.
- Accept/reject revision.
- Recompose full video.

Do not attempt to build a full professional video editor initially.

### 17.8 Tool registry/settings

Show:

- Enabled tools.
- Disabled tools.
- Tool permissions.
- Required environment variables.
- Test connection buttons.
- Tool call logs.
- Per-curriculum tool policy.

### 17.9 Voice settings

Show:

- TTS provider.
- Selected voice.
- Preview sample text.
- Save voice profile.
- Pronunciation dictionary.
- Voice consistency status.

### 17.10 YouTube settings

Show:

- Connected account/channel.
- OAuth status.
- Requested scopes.
- Disconnect button.
- Default visibility.
- Default playlist behavior.
- Publishing defaults.

### 17.11 Admin/storage

Show:

- Disk usage.
- Largest curricula.
- Render cache size.
- Export curriculum zip.
- Import curriculum zip.
- Rebuild SQLite index.
- Clean failed/rejected renders.
- Database integrity/check status if implemented.

---

## 18. UX interaction examples

### 18.1 Create a curriculum

User:

```text
I want to learn NAT and home networking from first principles.
```

Agent:

- Creates curriculum.
- Asks or infers audience/depth.
- Generates table of contents.
- Proposes playlist.
- Creates content orders.

### 18.2 Generate videos

User:

```text
Generate the first three videos in this playlist as drafts.
```

Agent:

- Creates lesson plans.
- Creates scenes.
- Generates narration and visual specs.
- Renders scene chunks.
- Synthesizes voice.
- Composes draft videos.
- Marks them as needing review.

### 18.3 Revise scene with mic

User selects scene 4 and records:

```text
This part is too abstract. Show the NAT table filling in one row at a time and slow down the narration.
```

System:

- Transcribes audio.
- Attaches transcript to scene 4.
- Agent parses edit intent.
- Revises scene script/visual spec.
- Renders new scene preview.
- User accepts or rejects.

### 18.4 Publish to YouTube

User:

```text
Prepare this playlist for YouTube upload.
```

Agent:

- Generates titles.
- Generates descriptions.
- Generates thumbnails.
- Creates chapters.
- Runs publish readiness checks.
- Shows package for approval.
- Uploads as private after user approval.

---

## 19. YouTube description style

Descriptions should be consistent across a playlist.

Support a reusable description template.

Example sections:

```text
What you’ll learn
Chapters
Source notes / references
Related videos in this playlist
Generated/edited with this local learning studio
```

The user should be able to edit the template per curriculum.

Chapters should come from scene boundaries where practical.

---

## 20. Testing expectations

### 20.1 Playwright

Use Playwright tests from the start where practical.

The coding agent should be able to use Playwright CLI ad hoc to click around and validate behavior during development.

Also include programmatic Playwright tests for core flows.

### 20.2 Suggested end-to-end tests

Include tests for:

```text
create curriculum
generate table of contents with mocked model
create lesson
create scene
render mocked scene
record or submit text feedback
regenerate chunk
accept revision
generate thumbnail with mocked provider
prepare YouTube publishing package
mock YouTube upload flow
```

### 20.3 Mock external providers

Default tests should mock:

- OpenAI transcription.
- OpenAI image generation.
- Qwen3-TTS if not locally available.
- YouTube API.
- Local LLM endpoint if needed.

Real integration tests may be enabled by environment variables.

### 20.4 Backend tests

Include tests for:

- SQLite migrations.
- Artifact registration.
- Filesystem workspace creation.
- Job queue behavior.
- Tool permission enforcement.
- OAuth callback handling with mocks.
- Publish readiness gates.

---

## 21. Security and approval model

### 21.1 Secrets

Secrets should be configured through environment variables or local settings.

Examples:

```text
OPENAI_API_KEY
GOOGLE_OAUTH_CLIENT_ID
GOOGLE_OAUTH_CLIENT_SECRET
LOCAL_LLM_BASE_URL
QWEN_TTS_BASE_URL
```

The app should avoid injecting secret values into agent prompts.

### 21.2 Approval categories

Actions can be categorized:

```text
auto_allowed
confirm_expensive
confirm_external
confirm_destructive
confirm_publishing
```

Examples:

```text
render local preview → auto_allowed
generate thumbnail with paid API → confirm_external or governed by budget
delete rejected draft render → confirm_destructive, maybe batch allowed
upload private YouTube video → confirm_publishing
make playlist public → confirm_publishing
```

### 21.3 Agent tool access

The agent should call typed backend tools, not raw APIs.

Good:

```text
youtube_upload_video(publishing_package_id)
```

Bad:

```text
Agent gets raw OAuth token and constructs arbitrary YouTube API calls.
```

---

## 22. Import/export

### 22.1 Export curriculum

Support exporting a curriculum as a zip.

Options:

```text
include all artifacts
exclude render cache
include only accepted/final artifacts
include source snapshots
include YouTube metadata
```

### 22.2 Import curriculum

Support importing a curriculum zip later.

The app should rebuild SQLite metadata from imported files where possible.

### 22.3 Backup philosophy

Because workspaces are filesystem-first, backup should be simple:

```text
copy /data
or export selected curriculum
```

---

## 23. ADR requirements

### 23.1 Initial ADRs

The coding agent should create at least these ADRs:

```text
0001-use-vite-react-static-frontend.md
0002-use-python-fastapi-backend.md
0003-use-filesystem-volume-as-source-of-truth.md
0004-use-sqlite-as-rebuildable-index-and-job-store.md
0005-use-pydantic-ai-for-agent-runtime.md
0006-use-in-process-agent-runner-for-first-version.md
0007-use-tool-permission-model.md
0008-use-youtube-oauth-not-raw-credentials.md
0009-use-manim-and-ffmpeg-as-first-renderers.md
0010-require-human-approval-for-publishing.md
```

### 23.2 ADR template

```markdown
# ADR NNNN: Title

## Status
Accepted / Proposed / Superseded

## Context
What problem are we solving?

## Decision
What are we choosing?

## Consequences
What gets easier?
What gets harder?
What risks remain?

## Alternatives considered
What else did we consider?

## Revisit when
What future condition should cause us to reconsider?
```

---

## 24. First complete version scope

This is not a tiny MVP. Treat it as a **first complete vertical version**.

### 24.1 Must have

The first complete version should support:

- Static Vite frontend.
- FastAPI backend.
- Pydantic AI coordinator.
- Filesystem curriculum workspaces.
- SQLite metadata/job index.
- Create curriculum/course.
- Generate curriculum table of contents.
- Generate multiple lesson/video plans.
- Generate scene chunks.
- Render with Manim.
- Compose with FFmpeg.
- Choose and persist Qwen3-TTS voice profile.
- Generate voiceover.
- Preview video chunks.
- Submit text feedback on chunks.
- Submit microphone feedback on chunks.
- Regenerate selected chunks.
- Accept/reject revised chunks.
- Generate or bring thumbnail.
- Connect YouTube via OAuth.
- Create/update YouTube playlist.
- Prepare YouTube title/description/chapters.
- Upload video as private after approval.
- Publish readiness gates.
- Playwright smoke/e2e tests.
- ADRs.

### 24.2 Should have

- GPT Image 2 thumbnail provider.
- YouTube thumbnail upload.
- Playlist-level visibility management.
- Export curriculum zip.
- Admin disk usage page.
- Rebuild SQLite index action.
- Mocked provider test suite.
- Tool registry UI.

### 24.3 Could have later

- Captions upload.
- Auto-translation.
- Gemini-style whole-video review.
- HyperFrames renderer.
- HTML/Markdown scene renderer.
- Avatar teacher green-screen renderer.
- MCP tool integration.
- DBOS durable workflows.
- Local transcription provider.
- Multi-user auth.
- Analytics import from YouTube.
- Automatic Shorts generation.
- Full NLE-style timeline editing.

---

## 25. Open implementation choices for the coding agent

The coding agent may decide details such as:

- Exact Python package layout.
- Whether to use SQLAlchemy directly or SQLModel.
- SSE vs WebSocket vs polling for job events.
- Exact schema names.
- Exact tool class hierarchy.
- Exact React routing solution.
- Exact shadcn component usage.
- Exact visual style.

However, the coding agent should not change these without ADRs:

- Vite frontend.
- FastAPI backend.
- Pydantic AI.
- Filesystem-first storage.
- SQLite as local metadata/job index.
- Human approval for publishing.
- YouTube OAuth.
- In-process agent runner initially.
- Manim/FFmpeg first renderers.
- Qwen3-TTS first-class TTS support.

---

## 26. Instructions for UI design agent

The UI design agent should understand the whole product, but should not be over-constrained.

Design intent:

- Make the app feel like a creative studio.
- Show curriculum structure clearly.
- Make scene-level iteration obvious.
- Keep the central coordinator agent available.
- Make rendering and publishing status easy to understand.
- Keep dangerous/public actions clearly gated.
- Make microphone feedback feel natural.
- Avoid overwhelming the user with raw logs unless they ask.

Important UI concepts:

```text
curriculum
playlist
content order
lesson
scene
artifact
render job
voice profile
thumbnail
publishing package
quality gate
agent conversation
```

The UI should make it easy to answer:

- What am I making?
- What exists already?
- What still needs review?
- What failed?
- What can I regenerate?
- What will be uploaded?
- Is this public or private?
- What did the agent change?

The UI designer may propose better layouts, names, and flows.

---

## 27. Instructions for coding agent

The coding agent should start by:

1. Creating repo structure.
2. Creating ADR folder and initial ADRs.
3. Implementing FastAPI backend skeleton.
4. Implementing Vite React frontend skeleton.
5. Implementing SQLite migrations.
6. Implementing filesystem workspace creation.
7. Implementing basic curriculum CRUD.
8. Implementing job model.
9. Implementing artifact registration.
10. Implementing initial coordinator-agent stub.
11. Implementing mocked generation/render flow.
12. Implementing Playwright smoke tests.
13. Iterating toward real Manim/FFmpeg/TTS integrations.

The coding agent should not begin by building the hardest AI/media integrations first. Build the skeleton and artifact/job model first, then connect real tools.

---

## 28. Core north-star loop

The most important loop is:

```text
Create curriculum
→ generate table of contents
→ generate lessons and scenes
→ render chunks
→ watch preview
→ select weak section
→ dictate or type feedback
→ agent patches smallest artifact
→ rerender chunk
→ accept/reject
→ compose final video
→ prepare YouTube package
→ approve upload
→ upload as private
→ optionally make playlist public later
```

Everything in the architecture should serve this loop.

---

## 29. Final product summary

Build a self-hosted, filesystem-first instructional video studio with a static Vite frontend, Python/FastAPI backend, Pydantic AI coordinator, SQLite metadata/job index, Manim/FFmpeg rendering, Qwen3-TTS voice support, microphone feedback, scene-level regeneration, thumbnail generation, and YouTube OAuth publishing.

The app should be simple where possible, extensible where necessary, and safe around public/destructive actions.

The first complete version should be ambitious enough to create and publish private YouTube-ready instructional videos, while preserving clean architecture for future renderers, review tools, local models, captions, translations, Shorts, and more advanced agent workflows.