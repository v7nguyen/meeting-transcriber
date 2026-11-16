# Meeting Transcriber — macOS Architecture

## Overview

Native SwiftUI menu bar application that orchestrates meeting detection, recording, transcription, diarization, and protocol generation. Runs a background watch loop (`WatchLoop`) polling for active meetings and implementing a complete end-to-end pipeline.

**Key pattern:** Observable state models (`@Observable`) with PipelineQueue for decoupled post-processing.

---

## Pipeline

```
                ┌──────────────────────────────────────────────────┐
                │           MeetingTranscriberApp (@main)          │
                │   SwiftUI: MenuBarExtra + Settings + Naming      │
                └──────────────────────────┬───────────────────────┘
                                           │ owns
                ┌──────────────────────────▼───────────────────────┐
                │            AppState (@Observable @MainActor)     │
                │   concern controllers, settings, engines         │
                └────────┬─────────────────────────────────┬───────┘
                         │                                 │ optional
                         │                                 │ (#if !APPSTORE
                         │                                 │  + toggle/env)
                         │                                 ▼
                         │                  ┌──────────────────────────┐
                         │                  │  DebugRPCServer          │
                         │                  │  127.0.0.1:9876          │
                         │                  │  /state /healthz /metrics│
                         │                  │  /screenshot             │
                         │                  │  /action/openSettings    │
                         │                  │  /action/closeSettings   │
                         │                  │  /v1/* (automation API)  │
                         │                  │  Bearer-token + Origin   │
                         │                  │  reject. Driven by mt-cli│
                         │                  └──────────────────────────┘
                         │
                         ▼
                ┌─────────────────────────────────────────────────┐
                │           WatchLoop (@MainActor)                │
                │   idle → watching → recording → watching        │
                └────┬───────────────┬────────────────────┬───────┘
                     │ polls         │ starts/stops       │ enqueues
                     ▼               ▼                    ▼
        ┌─────────────────────┐  ┌────────────────┐  ┌────────────────────┐
        │ MeetingDetecting    │  │ DualSource-    │  │   PipelineQueue    │
        │  • MeetingDetector  │  │   Recorder     │  │   (@MainActor)     │
        │    (CGWindowList +  │  │                │  │                    │
        │     regex)          │  │  AudioTapLib   │  │  Sequential job    │
        │  • PowerAssertion-  │  │  (CATap +      │  │  processing →      │
        │    Detector (IOKit, │  │   AVAudioEng.) │  │  see breakdown     │
        │    sandbox-safe)    │  │  + AudioMixer  │  │  below             │
        │  • MicInputDetector │  │                │  │                    │
        │    (CoreAudio proc) │  │                │  │                    │
        └─────────────────────┘  └────────────────┘  └─────────┬──────────┘
                                                               │
        PipelineQueue per-job processing:                      │
        ┌──────────────────────────────────────────────────────▼─────────┐
        │ 1. Resample to 16 kHz mono (AudioMixer; AVAsset / ffmpeg fb)   │
        │ 2. (opt) FluidVAD silence-trim + timeline remap                │
        │ 3. Transcribe via active engine                                │
        │      └─ TranscribingEngine: WhisperKit | Parakeet             │
        │         (dual-source: each track separately, then merge)       │
        │ 4. (opt) Diarize via FluidDiarizer                             │
        │      └─ Mode: .offline | .sortformer | .nemotron               │
        │      └─ Dual-source: app + mic diarized separately,            │
        │         IDs prefixed R_ (remote) / M_ (mic), then merged       │
        │ 5. SpeakerMatcher: cosine match against speakers.json          │
        │      (centroid + recent-FIFO, threshold 0.40, margin 0.10)     │
        │ 6. Speaker naming UI (suspended via CheckedContinuation)       │
        │ 7. Assign speakers to transcript by temporal overlap           │
        │ 8. Save transcript (.txt)                                      │
        │ 9. Protocol generation                                         │
        │      └─ ProtocolProvider: .claudeCLI | .openAICompatible | .none│
        │ 10. Save protocol (.md, transcript appended)                   │
        │ 11. Job done; export to Notion in the background (optional)    │
        └────────────────────────────────────────────────────────────────┘
```

State writes to `AppPaths.dataDir`; IPC + queue snapshots to `ipcDir`.

---

## Source Files

### App Entry & UI

| File | Role |
|------|------|
| `AppLauncher.swift` | The actual `@main` — a process entry point that makes the one pre-launch diversion decision (e.g. `--localvqe-selftest`) before `MeetingTranscriberApp`/`AppState` construct, since a conforming `App` type cannot intercept its own launch |
| `MeetingTranscriberApp.swift` | UI shell — SwiftUI scenes, windows, NSOpenPanel, NSWorkspace. Observes `.showSettings` / `.closeSettings` / `.showSpeakerNaming` notifications for RPC- and pipeline-driven scene control; logs menu-triggered protocol-open and quit requests |
| `AppState.swift` | `@Observable @MainActor` composition root — wires the concern controllers (`engines`, `watching`, `pipeline`, `permissions`, `channelHealth`, `liveTranscription`, `rpcController`) and exposes the derived UI state (badge, status label) rather than owning it |
| `AppState+MicMute.swift` | Hoisted out of the menu-bar body, which has a type-check budget the analyze build enforces; also what takes the mic-silent tint off the icon once the user mutes |
| `AppState+Termination.swift` | What a quit waits for, as `AppState` answers it (see `TerminationFlush`) — reads `watching` and `pipeline` at quit time rather than capturing either earlier |
| `LivenessMarker.swift` | Process-lifetime marker (issue #703): written at launch, touched once a minute, removed by a clean AppKit quit; the next launch reads it as `PreviousExit` to tell a crash, kill or power loss from a quit, which the per-recording marker cannot do for an idle app |
| `PreviousExitNotice.swift` | Wording of the launch-time notification for a run that ended without a quit: the window in which nothing was watching for meetings |
| `MenuBarView.swift` | Menu bar dropdown (state, actions, meeting info) |
| `MenuStageProgressUpdater.swift` | Updates progress on the existing native menu item in place; rebuilding SwiftUI menu content on elapsed-time ticks or stage changes displaces Cancel's highlight |
| `QueueHeader.swift` | The heading and icon above the menu's job list, chosen from what the jobs are doing (working / waiting on the user / all finished / a failure among finished ones) — pure, like `BadgeKind.compute`, so the precedence lives in one place |
| `ProtocolOpener.swift` | Opens local protocols with the default handler and prefers an already-running Notion desktop app for Notion pages, falling back to the default handler |
| `MenuBarIcon.swift` | Renders the animated waveform icon + badge overlays (permission, record-only, channel-silent) |
| `AppPickerView.swift` | App picker sheet for manual recording of any running app |
| `AudioImportTypes.swift` | File types offered by the batch-import and voice-enrollment `NSOpenPanel`s — single source of truth so the ffmpeg-gated vs. natively-decoded format lists stay pinned and testable |
| `A11yID.swift` | Shared accessibility-identifier namespace — one constant per control, referenced by the view modifier, ViewInspector tests, and the `/ui/press` allowlist |
| `SettingsView.swift` | Settings window — `NavigationSplitView` shell (sidebar of tabs) hosting seven topic-grouped sub-views in `Sources/Settings/` |
| `Settings/GeneralSettingsView.swift` | Startup (Launch at login, Start watching when the app opens = `AppSettings.autoWatch`) · Mode (Record-only) · Apps to Watch (Teams/Zoom/Webex/Browser/WeChat/Tencent Meeting/FaceTime/WhatsApp/Slack) · Detection (Poll Interval, Grace Period) |
| `Settings/AudioSettingsView.swift` | Microphone device · VAD (enabled + threshold) · Per-Channel Indicator |
| `Settings/MicMuteHotkeyRow.swift` | Settings row that captures the shortcut toggling mic mute during a recording; re-reads availability on each capture so a shortcut disabled in System Settings meanwhile is still allowed, and explains why the last typed combination was refused |
| `Settings/TranscriptionSettingsView.swift` | ASR engine picker · engine-specific options · model status · Vocabulary section · Live transcription (PoC) toggle |
| `Settings/LegacyWhisperModelsSection.swift` | Storage section of Transcription settings listing WhisperKit models an earlier build left in `Documents`; shown only while there is something to clean up, nothing pre-checked since only the user knows whether another app still uses a copy |
| `Settings/LegacyWhisperModelsState.swift` | Backing state for that section: lists the leftover variants and moves the ones the user picked to the Trash after confirmation, off the main actor; the listing and copy live in `LegacyWhisperModelInventory` |
| `Settings/SpeakersSettingsView.swift` | Diarization · Speaker Naming (open the naming window automatically, how long unnamed meetings wait) · Mic Speaker Name · Known Voices · Recognition Stats · Experimental: Diarization Tuning |
| `Settings/OutputSettingsView.swift` | LLM provider · notes language · notify when meeting notes fail · output folder · recording retention · custom prompt |
| `Settings/OpenAIConnectionControls.swift` | Saved OpenAI-compatible connection picker, name field, Add/Delete |
| `Settings/OpenAIConnectionPrompts.swift` | State of the Add (name prompt) and Delete confirmation dialogs |
| `Settings/DraftTextField.swift` | A text field whose value reaches its owner on Return or when the field loses focus, not on every keystroke — needed wherever a typed value is synced into a live engine/Keychain on change |
| `Settings/Notification+SettingsWindow.swift` | Shared check for whether an `NSWindow.willCloseNotification` is the Settings window's own; closing it only hides it, so draft-owning views check for this in one place rather than relying on focus change or `onDisappear` |
| `Settings/ProtocolAPIKeyDrafts.swift` | What is being typed into the meeting-notes API key fields, held outside the Keychain until a field commits (see `DraftTextField`) — binding a field straight to the Keychain-backed setting meant every keystroke was a Keychain write |
| `Settings/WhisperKitCustomModelDrafts.swift` | What is being typed into the custom WhisperKit model fields, held outside `AppSettings` until a field commits — one object rather than three `@State` strings so a view test can see what "Load Model" actually reads |
| `Settings/AdvancedSettingsView.swift` | Permissions · Diagnostics |
| `Settings/DiagnosticExportStatus.swift` | What Settings → Advanced says after Export Diagnostics; an export that found no recent entry still writes a file (header + a notice why), so it reads as a success without reading like a full one |
| `Settings/AboutSettingsView.swift` | Version · bundle identifier · build date · ffmpeg status · project link · Updates |
| `Settings/HelpBadge.swift` / `Settings/SettingsHelp.swift` | Reusable "?" help-popover badge + its shared copy, used across Settings sections |
| `Settings/View+RecordOnly.swift` | `recordOnlyDisabled(_:)` view modifier — dims + disables the Transcription/Protocol/VAD/Diarization sections when record-only mode is on |
| `SpeakerNamingView.swift` | Speaker naming dialog after diarization |
| `NamingJobPicker.swift` | Menu of the meetings waiting for names, above the naming dialog once more than one waits (a menu so the content-sized window stays narrow) |
| `NamingGraceKey.swift` | Identity of one keyboard-grace window in the naming dialog — what counts as "a new grace window" (data revision + pending-job count), so the gate re-locks when another job steals focus |
| `NamingAnswerGate.swift` | Whether the naming dialog still takes an answer for the job it shows, and why the host refused the last one; an answer is handed over at most once per job, and a refused answer keeps the buttons live with `refusalMessage` explaining why |
| `SpeakerNamingRowState.swift` | Per-presentation state of the naming rows, keyed by speaker label (never row position) and held as a reference type so a row's write is observable from outside the view |
| `SpeakerNamingView+Snippet.swift` | Reads the sample audio for a naming row's playback snippet directly from the meeting's 16 kHz file for the given time span |
| `KnownVoicesView.swift` | Manage persisted speaker DB (rename, delete, merge) — embedded in `SpeakersSettingsView` |
| `RecognitionStatsView.swift` | Recognition stats display — aggregate counts from `recognition_log.jsonl` |
| `VoiceEnrollmentView.swift` | Voice enrollment sheet — seeds `speakers.json` from an existing audio file |
| `AppSettings.swift` | `@Observable` settings persisted to UserDefaults |
| `AppSettings+Computed.swift` | Values derived from stored `AppSettings` toggles, split out to keep `AppSettings.swift` under the line cap |
| `AppSettings+OutputDirectory.swift` | Everything derived from `customOutputDirBookmark`, split out to keep `AppSettings.swift` under the line cap |
| `AppSettings+Vocabulary.swift` | Custom-vocabulary file selection, sandbox bookmark resolution, and validation state — shared by both ASR engines |
| `AppSettings+OpenAIConnections.swift` | Add, select, rename and delete saved OpenAI-compatible connections |
| `AppSettings+TerminologyRules.swift` | Optional terminology-rules file: selection, bookmark, per-job read and the Settings status; the text area remains the default source |
| `AppSettings+WhisperKitModel.swift` | The WhisperKit model the settings ask for: a variant and where it comes from |
| `LegacyDefaultsMigration.swift` | One-shot carry-over of settings from the pre-rename bundle identifier, since `UserDefaults` is scoped per identifier |
| `LoginItemController.swift` | Launch at login via `SMAppService.mainApp`, behind `LoginItemRegistering` so tests use a fake; the status is read from macOS and never stored in `AppSettings` |
| `UpdateChecker.swift` | Checks GitHub releases for newer versions, drives the menu bar update badge |
| `UpdateInstallRoute.swift` | How an available update should reach this install — a Homebrew cask install upgrades via `brew upgrade` rather than dragging the release DMG over it, which would leave Homebrew's own record at the old version |
| `Settings/PickerLanguages.swift` | Language picker entries for WhisperKit and Parakeet language selectors |
| `LiveCaptionsState.swift` | `@Observable` live-captions state (per-channel hypotheses + finalised utterances) + RPC-wire types |
| `LiveCaptionsWarmup.swift` | The live-caption model warm-up, as Settings, the menu and `/state` show it — no controller, the time the controller was built, or no caption model loaded (with the reason) |
| `LiveCaptionsOverlay.swift` | SwiftUI caption-bar content (recent finals + per-channel hypotheses) hosted in `LiveCaptionsWindow` |
| `LiveCaptionsWindowController.swift` | Borderless click-through NSPanel hosting the caption overlay; Option-drag moves it by screen-coordinate deltas, clamped to the screen visible frame (see `architecture/transcription.md`) |
| `LiveCaptionsSize.swift` | Caption-bar size presets (Small/Medium/Large): font size paired with the fixed panel dimensions that fit it; `AppSettings.liveCaptionsSize` |
| `CaptionBarPlacement.swift` | Where the caption bar is placed each time it is shown or resized, over plain rects and attached-screen frames so every case can be pinned without a display |
| `CaptionDragSession.swift` | Which moves of the caption panel count as the user's drag and get saved, with the button state passed in so every case can be pinned without a mouse; screen-coordinate translation from the mouse and panel origins at drag start |
| `ProcessingStatsView.swift` | Read-only average per-stage processing durations from `stage_timing.jsonl` (Settings → Speakers) |

### Core Pipeline

| File | Role |
|------|------|
| `WatchLoop.swift` | Main orchestrator: detect → record → enqueue PipelineJob |
| `WatchLoop+Presentation.swift` | Title cleanup and the `TranscriberState` mapping, split out of `WatchLoop` for size |
| `WatchLoop+ManualRecording.swift` | The poll loop that decides when a manually started recording ends, split out of `WatchLoop` |
| `WatchLoopEndPolicy.swift` | Pure decision logic for `waitForMeetingEnd` (grace-period / max-duration) |
| `WatchLoop+MeetingEnd.swift` | How `WatchLoop.waitForMeetingEnd` finished: the meeting ended, hit the max duration, or watching was stopped, or the app moved on to a different call that is already live |
| `WatchLoopState.swift` | Value-type snapshot of `WatchLoop`'s observable fields (for tests and RPC) |
| `ManualRecordingMonitorPolicy.swift` | Pure decision logic for manual recording stop conditions (process-died vs max-duration) |
| `MeetingDetecting.swift` | `MeetingDetecting` protocol + `DetectedMeeting` model |
| `MeetingDetector.swift` | Window title polling, pattern matching, confirmation counting, cooldown. Not the production detector (that is `CompositeMeetingDetector` over `PowerAssertionDetector` + `MicInputDetector`); production uses only its `systemWindowList` to look up meeting titles |
| `MeetingTitleMatcher.swift` | Compiled idle/meeting title regex semantics for one `AppMeetingPattern` |
| `PowerAssertionDetector.swift` | IOKit power assertion–based meeting detection (sandbox-safe); carries the Chrome WebRTC pattern for browser meetings (issue #503) |
| `PowerAssertionDetector+Diagnostics.swift` | "Saw something that looks like a meeting and did not act on it" diagnostic, split out of `PowerAssertionDetector` (line-cap) |
| `MicInputDetector.swift` | Third `MeetingDetecting` strategy: watches which processes hold `kAudioProcessPropertyIsRunningInput` via the Core Audio process-object API — covers call apps (WeChat, Tencent Meeting, FaceTime, WhatsApp, Slack Huddles) with no reliable power-assertion signal; each app opt-in and off by default. Also carries Safari browser meetings (WebKit GPU process attributed to Safari), selected by Browser Web Meetings and consent-gated |
| `MeetingPatterns.swift` | Regex patterns for Teams, Zoom, Webex, browser (Chrome WebRTC) |
| `BrowserConsentPolicy.swift` | Pure decision logic for the browser-meeting "ask before recording" prompt — decline cooldown (issue #503) |
| `ConsentAnswer.swift` | Three-way outcome of a consent prompt (yes / no / unanswered) — kept distinct from a `Bool` so a decline and a timeout get different re-prompt cooldowns (issue #543) |
| `BrowserConsentReadiness.swift` | Whether a browser-meeting consent prompt can actually reach the user — polls `NotificationVisibility` since the prompt is itself a notification and a broken notification channel can't report its own brokenness |
| `ConsentPromptCoordinator.swift` | Coordinates an async yes/no recording-consent prompt: register pending decision by id, resolve once via answer or timeout |
| `WatchLoop+Consent.swift` | Browser-meeting consent gate, split out of `WatchLoop`; only patterns with `requiresRecordingConsent` reach it |
| `DualSourceRecorder.swift` | Orchestrates AudioTapLib capture + mic, mixes tracks |
| `DualSourceRecorder+TapPIDs.swift` | Which processes a recording taps, split out of `DualSourceRecorder.swift` to keep that file under the line cap — pure, static, tested on its own |
| `RecorderError.swift` | Why a recording could not start or produced nothing to process; its own file rather than the tail of `DualSourceRecorder.swift`, which is at the line cap — `WatchLoop` and `WatchingController` throw and match it too |
| `RecordingProvider.swift` | Protocol abstraction over `DualSourceRecorder` for mock injection in `WatchLoop` tests |
| `WatchLoop+RecordOnly.swift` | Record-only output branch (moves WAVs + writes `RecordingSidecar`), split out of `WatchLoop` |
| `WatchLoop+MicMute.swift` | Toggles the active recorder's mic mute and mirrors its reported mute state back onto `WatchLoop` |
| `WatchLoop+Quit.swift` | The parts of a quit that `WatchLoop` answers for without needing its private state: where a recording the quit stopped goes, and what a stop that recorded nothing means while quitting; stops and enqueues a recording a quit is finishing with the file work off the main actor |
| `ProtocolResumePolicy.swift` | Decides what the snapshot restore does with a job interrupted mid-run: resume from the saved transcript, just finish, or run in full. Keys on the interrupted stage, never on "a transcript exists", because stage 1 writes a draft without speaker labels |
| `AudioPersistencePolicy.swift` | Decides per finished-job source file whether to relocate it into the output folder or leave it in place (staging-dir recording vs. user-picked import) |
| `TimestampedSegment.swift` | A transcribed segment with timestamps and optional speaker label; every field is `var` on purpose so downstream passes (delay shift, VAD remap, block merge) adjust one or two fields on an existing value without silently resetting the ones they don't touch |
| `TranscriptNote.swift` | Puts a recording-level annotation at the top of a transcript, rendered through the one shared `[TimestampedSegment].transcriptText(note:)` call site every transcript is written through |
| `TranscribingEngine.swift` | `TranscribingEngine` protocol + `mergeDualSourceSegments` default impl |
| `WhisperKitEngine.swift` | WhisperKit transcription engine (99+ languages, ~1 GB model) |
| `WhisperDecodingClient.swift` | Narrow decode boundary used by `WhisperKitEngine` — production forwards to WhisperKit, tests capture the exact options passed |
| `WhisperKitEngine+ModelLoad.swift` | The model load's retry loop, split out because the engine's own file is at the length limit; retries while the requested model moved on during a pass |
| `WhisperKitEngine+Segments.swift` | Turns WhisperKit's decode output into transcript segments, pure so the failure rules and hallucination filter can be tested without a model |
| `WhisperKitLoadFailureReason.swift` | The one sentence a failed WhisperKit model load is reported with, in Settings and in the job's error text — worded here rather than taken from WhisperKit's own error description, since the likely errors carry a local path |
| `WhisperKitLocalSnapshot.swift` | Where WhisperKit keeps its downloaded models, and whether a complete copy of a variant is already present — the reason the local-first load path exists, since `WhisperKit.download` cannot serve an already-present model without the network |
| `WhisperKitModelMigration.swift` | Moves WhisperKit models an earlier build downloaded into `Documents/huggingface` over to the current download base, once per variant and process, off the main actor, so an existing install does not re-download a gigabyte |
| `WhisperKitModelOrigin.swift` | Where the files of a WhisperKit model variant come from; part of a model's identity next to its variant name, since a fine-tune usually keeps the folder name of the model it was trained from |
| `WhisperKitModelSource.swift` | The named steps `WhisperKitEngine.loadModel()` takes to reach a usable pipeline, so a test can observe which ran; production resolves every step against WhisperKit itself, local-first per `WhisperKitLocalSnapshot` |
| `LegacyWhisperModelInventory.swift` | A WhisperKit variant folder an earlier build left in `Documents/huggingface`: folder name, and what it occupies on disk (an iCloud-evicted file counts zero); finds and removes these leftovers |
| `WhisperVocabularyPrompt.swift` | Converts the shared custom-vocabulary file into a bounded WhisperKit decoder prompt (experimental, off by default) |
| `ParakeetEngine.swift` | NVIDIA Parakeet TDT v3 via FluidAudio (25 EU languages, ~50 MB, ~10× faster) |
| `ParakeetTokenGrouping.swift` | Pure token-grouping logic extracted from `ParakeetEngine` (testable) |
| `ParakeetVocabularyConfiguration.swift` | Identity of one Parakeet CTC vocabulary configuration (path + bookmark + file revision) and its preparation lifecycle |
| `VocabularyFileAccess.swift` | Resolves persisted sandbox access for the shared custom-vocabulary/terminology files, brackets reads with the matching security scope |
| `TerminologyNormalizer.swift` | Opt-in post-ASR canonical-spelling rules (`Canonical => variant \| variant`), engine-independent |
| `TranscriptionFailureScope.swift` | Whether a transcription failure belongs to the file being transcribed or to the engine — a dual-source job may keep one track when the other's failure is about that track alone (unreadable audio, a bad decode), never when the model itself can't load |
| `DualTrackViability.swift` | Which tracks of a dual-source recording carry enough audio to transcribe; keeps an empty track (under WhisperKit's 300 ms minimum) from throwing the whole job down with it |
| `DualTrackTranscription.swift` | What a dual-source job keeps from its two transcription passes, dropping a track that fails on its own file the same way `DualTrackViability` drops an empty one |
| `StreamingTranscriber.swift` | Per-channel live transcription actor (FluidVAD streaming → `engine.transcribeSamples` → partial/final captions) |
| `CaptionChannelFeed.swift` | A duration-bounded audio queue with one serialized consumer per channel, feeding the live-caption pipeline per track |
| `PipelineQueue.swift` | Decouples recording from post-processing, sequential job pipeline |
| `PipelineQueue+Stages.swift` | Per-stage job processing (transcribe → diarize → naming → protocol), split out of `PipelineQueue` (line-cap) |
| `PipelineQueue+Notion.swift` | Optional Notion export step and page tracking for protocol retries and re-exports |
| `NotionClient.swift` | Notion API client: page create and re-export, schema checks, request batching within Notion's limits, 429 backoff |
| `NotionModels.swift` | Request and response types for the Notion API |
| `MarkdownToNotionBlocks.swift` | Converts protocol Markdown and the transcript into Notion blocks |
| `NotionClientError.swift` | Failures of the Notion client, with fixed cause text shown in a job warning — never Notion's own response text, which can quote meeting content back |
| `NotionExportDeadline.swift` | Bounds one export as a whole, since the Notion client only limits each request and a replace of a long page is hundreds of them; gives up on the overall operation after the limit or cancellation, whichever comes first |
| `NotionExporting.swift` | Result of a meeting export to Notion: optional values the target database couldn't accept (named in the job warning, never silently dropped), and whether this run updated an existing page rather than creating a new one |
| `NotionReplacePlan.swift` | Decides what a replace of an existing Notion page has to write and delete, from its current root blocks alone — needed because an interrupted replace leaves old and new blocks mixed with no boundary marker |
| `NotionRequestBatches.swift` | Keeps each Notion request inside its documented limits so a long meeting doesn't fail identically on every retry; a block that still breaks a limit is refused up front rather than sent to fail the same way again |
| `JobOpenTarget.swift` | Decides what a finished job's Open action opens: the Notion page or a local file |
| `ProtocolOpener.swift` | Opens a job's target, preferring a running Notion desktop app |
| `Settings/NotionExportSettingsView.swift`, `Settings/NotionExportSettingsModel.swift` | Notion settings UI and its connection and schema state |
| `PipelineQueue+Recovery.swift` | Snapshot restore for `PipelineQueue` |
| `PipelineQueue+MeetingNotes.swift` | Retrying meeting notes that failed, from the saved transcript (per job and all at once), and notes-only jobs for transcripts the user picks |
| `PipelineQueue+Retention.swift` | The queue's side of recording retention: records what stage 3 moved, marks a recording due once its output is written, runs passes with the tracks its unfinished jobs hold |
| `PipelineController+Retention.swift` | When retention passes run (launch, hourly, after a rebuild, once the settings settle), and the pass before a queue is built |
| `RecordingRetention.swift` | Runs retention passes: decides on the main actor, deletes off it via `RecordingFileDeleter`, rechecks each delete |
| `RecordingRetentionPolicy.swift` | Pure decision per registry entry: delete, keep (with the reason) or drop |
| `RecordingRetentionRegistry.swift` | The list of relocated recordings retention may delete (`retained_recordings.json`), the only thing the cleaner acts on |
| `PipelineQueue+OrphanRecovery.swift` | Orphaned-recording recovery for `PipelineQueue`: the staging scan and the jobs it creates |
| `StagingRecoveryGate.swift` | One staging recovery (header repair, crash re-mix, temp cleanup) at a time in this process, judging file ages from when it was asked for — two overlapping recoveries on one crashed stem could otherwise end with a mic-only mix over the complete one |
| `PipelineQueue+EchoBleed.swift` | Warns when a dual-source recording carries the same speech on both tracks (loudspeaker bleeding into the microphone) — measurement half of the echo mitigation (issue #581) |
| `PipelineQueue+EchoCancellation.swift` | Removes the far end from the microphone track before anything reads it — cancellation half of the echo mitigation, the other branch of `EchoRemedy` |
| `EchoBleedDetector.swift` | Decides `.notMeasured` / `.clean` / `.affected` for a dual-source recording, from per-10s-window envelope correlation between the two tracks |
| `EchoSegmentClassifier.swift` | Per microphone segment, decides whether the app track's energy explains it (`.echoOnly`, suppressed from the transcript) or the local speaker also spoke (`.mixed`/`.ownVoice`, kept) |
| `EchoEmbeddingQuarantine.swift` | Holds mic-track speaker embeddings back from `speakers.json` on an echo-affected recording, so a voice learned from contaminated audio is never folded into a stored centroid |
| `EchoRemedy.swift` | Decides which of the two echo remedies (cancellation vs. transcript dedup) a recording gets — they do not compose |
| `EchoCancelling.swift` | Abstraction seam for acoustic echo cancellation; `LocalVQECanceller` is the concrete implementation |
| `EchoCancellationSelfCheck.swift` | Judges a cancellation run from the canceller's own per-window report — a difference between far-end and no-far-end windows, not a level, so a run that quietly removed nothing is caught |
| `EchoFrameChunking.swift` | Pure hop arithmetic for streaming a signal through LocalVQE's fixed-256-sample-hop frame API |
| `LocalVQECanceller.swift` | `EchoCancelling` implementation over the vendored LocalVQE static library (`CLocalVQE`) — owns the C context lifecycle, drives the streaming frame API hop by hop |
| `LocalVQECanceller+File.swift` | File-to-file half of `LocalVQECanceller` — the shape the pipeline calls |
| `LocalVQEModel.swift` | Resolves the bundled LocalVQE AEC model (`localvqe-*.gguf`) the canceller runs against; refuses to silently fall back when an override path is missing |
| `LocalVQESelftest.swift` | Hidden `--localvqe-selftest` CLI probe proving the statically linked library resolves its compute backend from inside a signed `.app` bundle (`#if !APPSTORE`) |
| `PipelineJob.swift` | Pipeline job model (waiting → transcribing → diarizing → generatingProtocol → done) |
| `ArtifactRenamePlan.swift` | Plans the rename of one job's on-disk artifacts from one stem to another (the "use the AI title for filenames" feature) — pure, every filesystem fact passed in, so the decision is testable without a queue and the executor only runs what this returns |
| `PipelineSnapshot.swift` | Pure I/O helpers for persisting `PipelineQueue` jobs to disk (atomic rename) |
| `PipelineSnapshotStore.swift` | The one writer of a pipeline snapshot file per directory, so a queue rebuild that replaces one still-writing queue with another can't race two writers on the same `pipeline_queue.json` |
| `PipelineEventLog.swift` | Append-only JSONL log of `PipelineQueue` job state transitions |
| `StageTimingStats.swift` | Per-stage wall-clock duration tracking, backs `ProcessingStatsView` |
| `ProcessedRecordingsLedger.swift` | File-backed ledger of mix-file paths that completed the pipeline (dedupes recovery re-processing) |
| `InFlightRunRegistry.swift` | Process-wide record of currently-executing pipeline runs, so a second `PipelineQueue` built while the first is still working can't double-start the same job/audio (issue #558) |
| `SnapshotWriterActor.swift` | Actor isolating pipeline queue snapshot writes (prevents main-actor stalls on macOS 26 rename deadlock) |
| `SpeakerNamingSession.swift` | Collaborator `PipelineQueue` calls for the speaker-naming work still owned by the queue, split out for size |
| `SpeakerNamingSessionDelegate.swift` | The delegate protocol the naming session reaches the queue through, split out of `SpeakerNamingSession` for size |
| `SpeakerNamingSession+Late.swift` | Late-confirm and late re-diarization paths for speaker naming |
| `SpeakerNamingRefusal.swift` | Why the naming session did not take an answer from the naming dialog; the dialog shows the message and stays usable instead of going dead until reopened |
| `SpeakerNamingSession+Handler.swift` | The injected naming handler standing in for the dialog in tests and headless runs, split out of `SpeakerNamingSession+Late.swift` to keep both files under the lint cap |
| `SpeakerNamingData.swift` | `PipelineQueue.SpeakerNamingData`/`.Segment`/`SpeakerNamingResult` value types, nested under `PipelineQueue` for source compat |
| `StaleNamingTimeout.swift` | How long a meeting may wait for speaker names before the launch-time cleanup accepts the automatic names (1 day · 7 days · never) |
| `SpeakerNamingStore.swift` | Disk persistence for a job's speaker-naming sidecars, keyed by per-job slug; pure I/O, no queue state |
| `SpeakerKey.swift` | Speaker identity in the dual-track pipeline: raw diarizer id + track; single home for the `R_`/`M_` prefix convention |
| `SpeakerSlotRename.swift` | Renames the speaker slot of rendered transcript lines (`[MM:SS] Speaker: text`) from a diarization-label-to-confirmed-name mapping |
| `NamingWindowPolicy.swift` | Pins the "Name Speakers" window so it stays available while the user works in other apps (issue #504) |
| `LiveTranscriptionController.swift` | Wires the per-channel live-caption pipeline (streaming or re-transcribe) to both `DualSourceRecorder` sinks (mic + app), feeds `LiveCaptionsState` |
| `LiveTranscriptionController+Nemotron.swift` | Nemotron streaming-pipeline construction, split out for size; injectable factory for testing the build + model-load-failure fallback |
| `LiveTranscriptionCoordinator.swift` | `@Observable` coordinator: builds + arms `LiveTranscriptionController`, feeds `LiveCaptionsState` |
| `LiveCaptionPipeline.swift` | Per-channel live captioning strategy protocol (English EOU streaming \| Nemotron streaming \| WhisperKit/Parakeet re-transcribe) |
| `LiveCaptionsGate.swift` | Pure decision logic for live captions routing: master toggle + explicit engine language (`en` → English EOU streaming, a language in `LiveCaptionLanguages` → Nemotron streaming, auto-detect or any other language → re-transcribe or none) — shared by `AppState`, coordinator, and controller. Overlay visibility is a separate UI AND (`overlayVisible`); it is not a strategy input |
| `LiveCaptionLanguages.swift` | Nemotron tokenizer training coverage used to route live captions: tags are region-qualified (`<de-DE>`) but dropped where the tokenizer has only one dialect, since language pickers produce bare codes |
| `EouStreamingCaptionSession.swift` | English low-latency streaming caption session via FluidAudio Parakeet EOU, backed by `UtteranceRingBuffer` |
| `NemotronStreamingCaptionSession.swift` | Multilingual (non-English) low-latency streaming caption session via FluidAudio Nemotron |
| `NemotronAsrManager.swift` | Production seam wrapping FluidAudio's Nemotron model + Silero VAD for `NemotronStreamingCaptionSession` |
| `StreamingCaptionSplitter.swift` | Text-only boundaries over a prefix-monotonic cumulative transcript; never resets the decoder |
| `UtteranceRingBuffer.swift` | Rolling 16 kHz sample buffer addressable by absolute ms timestamp (feeds streaming caption sessions) |
| `AsyncSerialGate.swift` | First-come-first-served async mutual exclusion that suspends waiters instead of blocking a thread; serialises Nemotron 3 runs and model loads |
| `ModelWarmupQueue.swift` | Serial FIFO gate for model warm-up loads — avoids compiling/loading the ASR engine and live-caption models concurrently at launch |
| `EngineModelState.swift` | App-owned model lifecycle state for a `TranscribingEngine`, decoupled from any ASR vendor's own enum |
| `EngineController.swift` | `@Observable @MainActor` engine selection + model lifecycle controller (language/vocabulary sync, preload) |
| `PipelineController.swift` | `@Observable` controller owning `PipelineQueue` lifecycle (wired by `AppState`) |
| `PipelineController+MeetingNotes.swift` | Generate Meeting Notes from Transcript: refuses without a notes provider, notifies about skipped files |
| `PipelineController+Notion.swift` | Resumes a saved job's Notion export or failed meeting notes on the first queue a user action builds, since any other saved job keeps waiting for that action and would be resumed by building the queue anyway |
| `PipelineController+ProductionEnvironment.swift` | How `QueueEnvironment` is wired in production, with the staging-folder recovery handed to each new queue — split out of `PipelineController.swift` to keep it under the line cap, a pure move with no behavioural difference |
| `PipelineController+Quit.swift` | The pipeline side of a quit (see `TerminationFlush`): whether a snapshot write is still owed, and waiting for it — every queue this controller builds writes through the one `PipelineSnapshotStore` for that file |
| `TranscriptPanelDelegate.swift` | Enables only `.txt` files in the Generate Meeting Notes from Transcript panel |
| `OutputDirectoryResolver.swift` | Decides a recording's output folder at the two seams that capture it (queue build, record-only write); notifies once per episode when the chosen folder cannot be reached and the default stands in |
| `WatchingController.swift` | `@Observable` controller owning `WatchLoop` lifecycle (wired by `AppState`) |
| `WatchingController+Derived.swift` | `isWatching` / `isRecording` / `isManualRecording`, split out of `WatchingController` for size |
| `WatchingController+Detectors.swift` | Which detection strategies auto-watch runs, and how the "Apps to Watch" toggles filter them — line-cap split, pure and settings-driven |
| `WatchingController+WatchControl.swift` | The `/v1/watch` control surface: meeting watching as an idempotent resource a remote caller can drive |
| `WatchingController+RecordControl.swift` | The `/v1/record` control surface: microphone-only recording as an idempotent resource a remote caller can drive |
| `WatchingController+CaptionFlush.swift` | Starts the live-caption flush when a recording ends, its own file because `WatchingController.swift` sits near the line cap; not run on a call split, since the next call binds fresh caption feeds in the same main-actor stretch |
| `WatchingController+MicMute.swift` | Registers the mic-mute global hotkey against `WatchLoop.toggleMicMute()`, re-syncing whenever the hotkey or mute capability changes |
| `WatchingController+Quit.swift` | The watching side of a quit (see `TerminationFlush`): stop watching and hand whatever is recording to the pipeline before the snapshot is flushed; waits for an in-flight manual start before stopping it |
| `WatchingController+RecorderFactory.swift` | The recorder factory `WatchLoop` is handed: a fresh, freshly configured recorder per recording — its own file because `WatchingController.swift` is at the line cap |
| `WatchStatusDTO.swift` | Wire shape for `GET`/`POST /v1/watch` — the meeting-watching lifecycle as a small, stable projection |
| `RecordStatusDTO.swift` | Wire shape for `GET`/`POST /v1/record` — the microphone-recording lifecycle, same audience as `WatchStatusDTO` (Stream Deck key, Shortcut, shell script) |
| `WavHeaderRepair.swift` | Repairs unfinalized WAV files from crash-interrupted recordings (RIFF/data chunk size fix) |
| `FluidDiarizer.swift` | On-device speaker diarization via FluidAudio CoreML/ANE |
| `FluidDiarizer+SortformerEmbeddings.swift` | Post-hoc WeSpeaker embedding extraction for the Sortformer and Nemotron 3 modes — overlap-excluded masks feed `SpeakerMatcher` (DiariZen-style hybrid) |
| `FluidDiarizer+Nemotron.swift` | Nemotron 3 mode: the streaming loop that feeds a recording through `Nemotron3Diarizer` in 60 s pieces, one run at a time per instance; the model load sits behind an injectable seam and a process-wide load gate |
| `FluidDiarizer+NemotronModelLoad.swift` | The real Nemotron 3 model load (HuggingFace download, CoreML load on CPU + Neural Engine), the default of that seam |
| `SpeakerMatcher.swift` | Speaker embedding DB + cosine similarity matching |
| `SpeakerMatcher+Logging.swift` | Forensic match-decision logging (pseudonymized speaker names via `String.pseudonymized`) |
| `SpeakerMatcher+Editing.swift` | Rename, delete and merge for Known Voices and the debug RPC, each through `mutateDB` so an edit against an unusable DB answers `notSaved` rather than reading as done |
| `SpeakerMatcher+UnreadableDB.swift` | Tells a missing `speakers.json` apart from one that exists but can't be used, and reports whether a change actually reached disk — a missing file starts empty, a present one must never be silently overwritten |
| `LiveSpeakerMatcher.swift` | Actor for real-time speaker matching per finalized live-caption utterance; same WeSpeaker CoreML model as batch path; caches mask frame count in `UserDefaults` for fast cold start |
| `StoredSpeaker.swift` | Codable speaker DB entry model (centroid + FIFO embeddings + metadata) |
| `DiarizationProcess.swift` | Diarization result types, DiarizationProvider protocol, speaker assignment (standard + dual-track) |
| `ProtocolGenerator.swift` | Shared protocol utilities: `ProtocolGenerating` protocol, prompts, file I/O, `ProtocolError` |
| `MeetingNotesFailure.swift` | The kind of a meeting-notes failure, decided from the `ProtocolError` case, with the notification's fixed sentence and the sign-in hint |
| `ClaudeCLIProtocolGenerator.swift` | Claude CLI subprocess protocol generation (`#if !APPSTORE`) |
| `ClaudeCLIProtocolGenerator+Subprocess.swift` | One run of the CLI: its pipes and bounded lifetime — stdout/stderr read through `ProcessOutput`, the prompt written to stdin |
| `ClaudeCLIProtocolGenerator+SessionPersistence.swift` | Keeps the meeting out of the Claude CLI's own session store — left alone the CLI saves every run, transcript included, as a session under `~/.claude/projects/...`, a second unmanaged copy of the meeting |
| `ClaudeCLIProtocolGenerator+ToolRestriction.swift` | Starts the CLI with no tools and no MCP servers — the prompt carries the untrusted meeting transcript, and the user's full tool set and MCP servers would otherwise let a sentence said in the meeting act on them |
| `OpenAIProtocolGenerator.swift` | OpenAI-compatible API protocol generation (Ollama, LM Studio, etc.) |
| `OpenAIConnection.swift` | One saved OpenAI-compatible connection (key in the Keychain) |
| `RecordingSidecar.swift` | Metadata sidecar written next to recordings in record-only mode |
| `RecordingFileSuffix.swift` | Filename suffix constants for dual-source recordings (`_app.wav`, `_mic.wav`, `_mix.wav`) |
| `MicMuteLog.swift` | The mute times of a recording still in progress, on disk beside its tracks; `stop()` hands them over in `RecordingResult` and removes the file, so one still present means the recording never stopped |
| `SilentRecordingMonitor.swift` | Pure state machine detecting fully-silent recordings (both channels below threshold) |
| `ChannelHealthMonitor.swift` | Pure state machine for per-channel asymmetric silence detection (one channel live, other dead) — levels + hysteresis, drives the menu-bar tint |
| `ChannelHealthController.swift` | `@Observable` controller polling channel dBFS levels and driving `ChannelHealthMonitor` |
| `ChannelHealthController+Alerts.swift` | The notification copy `ChannelHealthController` posts: title, body and Focus behaviour per capture fault, plus the body for a recording that stayed silent — pure static builders kept apart so the wording reads in one place and the controller file stays under the length cap |
| `ChannelHealthController+LogLines.swift` | The diagnostics lines `ChannelHealthController` writes, built as pure strings so their content is testable and the controller file stays under the length cap |
| `ChannelFaultMonitor.swift` | Pure state machine deciding whether a channel has stopped delivering, from per-buffer ages rather than levels (issue #614) — drives the "Capture Channel Silent" notification, deliberately separate from the level-based tint |
| `PairedImportPanelDelegate.swift` | `NSOpenPanel` delegate + accessory view for paired dual-source file import |
| `PairedRecordingResolver.swift` | Groups recording URLs into dual-source groups (app + mic pairs, singletons) for reimport |
| `AudioCapturing.swift` | The part of `AudioTapLib.AudioCaptureSession` one recording drives — the consumer-side protocol `DualSourceRecorder` depends on |
| `DualSourceRecorder+BuildRecording.swift` | Turning a finished capture session into the files the pipeline consumes, split out of `DualSourceRecorder` |
| `MicDelayNormalisation.swift` | Turns a microphone that started before the app tap back into the alignment every downstream consumer expects: `AudioCaptureResult.micDelay` is mic-minus-app, so opening the microphone first (the issue #693 repair) makes it negative on every dual-source recording |
| `RecordingSource.swift` | What a single recording captures (app PID + mic on/off), replacing an untyped tuple |
| `ManualRecordingInfo.swift` | Info about a manually started recording session, as opposed to one a detector started |
| `ManualRecordingRequest.swift` | What the user asked `WatchingController` to record by hand (app-picker vs. microphone-only) |
| `AppPickerStartState.swift` | Whether the app picker can start a recording, and if not, why |
| `MicrophoneRecordingAvailability.swift` | Whether the menu bar's "Record Microphone" item can start a recording, and if not, why |
| `ConsentDenyList.swift` | Persisted "Never for this app" browser-meeting consent denials (issue #503 follow-up), checked before the decline cooldown |

### Audio Processing

| File | Role |
|------|------|
| `AudioMixer.swift` | Resampling, mixing, echo suppression, WAV I/O |
| `AudioMixer+AssetFallback.swift` | Tier 2 of `AudioMixer.loadAudioAsFloat32`, split out to keep that file under the `file_length` limit — decodes the first audio track through `AVAssetReader` as a mono fallback |
| `AudioMixer+Streaming.swift` | Decodes and resamples incrementally so long imports never need one buffer for the whole recording; `AVAudioPCMBuffer` stores byte capacity as `UInt32`, which a four-hour 48 kHz stereo recording exceeds even though its frame count still fits in `AVAudioFrameCount` |
| `AudioConstants.swift` | Shared audio pipeline constants (target sample rate) |
| `FFmpegHelper.swift` | ffmpeg CLI detection + 16 kHz mono WAV conversion fallback for file-import formats AVAsset can't decode |
| `FluidVAD.swift` | VAD preprocessing via FluidAudio Silero v6 — silence trimming + `VadSegmentMap` timeline remapping |
| `LiveAudioResampler.swift` | Streams live `LiveAudioBuffer` through `AVAudioConverter` → 16 kHz mono Float32 (feeds `StreamingTranscriber`) |
| `SampleRateDriftDetector.swift` | Watches actual vs declared CATap sample rate (catches USB hot-plug + HFP↔A2DP renegotiation drift) |
| `tools/audiotap/Sources/AppAudioCapture.swift` | CATapDescription + IOProc → FileHandle |
| `tools/audiotap/Sources/AppAudioCapture+PIDTranslation.swift` | Translates PIDs to CoreAudio `AudioObjectID`s (multi-process tap for Electron apps like Teams 2.x) |
| `tools/audiotap/Sources/AppAudioCapture+DebugLogging.swift` | Per-buffer dBFS/RMS logging helpers extracted from `AppAudioCapture` (line-cap split) |
| `tools/audiotap/Sources/AppAudioCapture+LiveSink.swift` | Live-buffer forwarding from CATap IOProc into `LiveAudioBuffer` sinks (line-cap split) |
| `tools/audiotap/Sources/AppAudioCapture+AggregateDescription.swift` | The CFDictionary describing the private aggregate device wrapping a process tap (line-cap split from `AppAudioCapture`) |
| `tools/audiotap/Sources/AppAudioCapture+Restart.swift` | Output-device-change restart path: off-main-queue, generation-tagged, deadline-bounded attempts (issue #588; line-cap split) |
| `tools/audiotap/Sources/AppAudioCapture+RateQueries.swift` | The sample-rate property queries and priority ladder between them, split out of `AppAudioCapture.swift` to stay under the line cap |
| `tools/audiotap/Sources/AppAudioCapture+SilentTrackDiagnostics.swift` | Log call sites for the silent-track instrumentation (issue #672), split out to stay under the line cap and keep the wording and privacy annotations together |
| `tools/audiotap/Sources/AppAudioCapture+SilentTrackWatchdog.swift` | The opt-in silent-track watchdog (issue #672 part 2): rebuilds the tap and aggregate through the device-change restart path once the app track has sat at exact zeros for `SilentTrackWatchdogPolicy`'s trigger window while buffers keep arriving |
| `tools/audiotap/Sources/AppAudioCapture+TapBuffer.swift` | Which buffer of the aggregate's input carries the process tap (issue #803) — the aggregate's own sub-device inputs arrive first when it has any, ahead of the tap's buffer |
| `tools/audiotap/Sources/AppTapSession.swift` | Owns one tap attempt's HAL resources (tap, aggregate device, IOProc) and their release ordering, injectable for testing without hardware |
| `tools/audiotap/Sources/AggregateRunState.swift` | What the aggregate device says about itself and which output device it was bound to (issue #693); `AudioDeviceStart` returning `noErr` says the object exists, not that IO is running |
| `tools/audiotap/Sources/NoFirstBufferProbeSchedule.swift` | When an app-audio capture that has delivered nothing is probed, and what the resulting diagnostic line says — a fourth probe point because the three issue #693 shipped with can't answer the question on their own |
| `tools/audiotap/Sources/MicCaptureHandler.swift` | AVAudioEngine → WAV |
| `tools/audiotap/Sources/MicCaptureHandler+Restart.swift` | Mic-side device-change restart path: off-main-queue, generation-tagged, deadline-bounded attempts (issue #588; same pattern as `AppAudioCapture+Restart`) |
| `tools/audiotap/Sources/MicCaptureHandler+DebugFault.swift` | Debug fault injection for the e2e lanes (issues #379, #724/#706), split out to keep `MicCaptureHandler.swift` under the line cap; inert unless a `DebugTapFault` was injected |
| `tools/audiotap/Sources/MicCaptureHandler+FirstStart.swift` | The first engine start under a deadline (issue #778) — the call that brings up an engine can wedge inside AVFAudio and never return, and the first start used to run unbounded on the main queue, unlike a restart |
| `tools/audiotap/Sources/MicCaptureHandler+Mute.swift` | Silences or restores the track from the next buffer on without reopening the input, since reopening is what can flip a Bluetooth headset's profile and wedge (issues #693, #588) |
| `tools/audiotap/Sources/MicMuteTimeline.swift` | One stretch of the microphone track the user muted, in seconds of that track |
| `tools/audiotap/Sources/MicCaptureHandler+Progress.swift` | Whether the microphone capture is getting anywhere, read by comparing the last buffer's mach time against the current epoch's start, since a bare age can't tell "nothing since I armed" from "nothing ever" |
| `tools/audiotap/Sources/MicCaptureProgressPolicy.swift` | What a stalled microphone capture should do next: nothing yet, rebuild the engine once the deadline passes, or release it and tell the user once the whole budget has run out |
| `tools/audiotap/Sources/MicCaptureStall.swift` | The microphone's stall as the capture layer reports it (issues #724, #706): released after going without audio for the whole budget, as two values since whether it is released now and whether there is a stall to tell the user about don't clear together |
| `tools/audiotap/Sources/MicEngineSession.swift` | Everything mic capture that touches `AVAudioEngine`/CoreAudio, behind a protocol so `MicCaptureHandler` is testable without hardware; one session = one engine lifetime, discarded and rebuilt on restart |
| `tools/audiotap/Sources/MicDevicePinOutcome.swift` | What came of pointing a mic engine's input unit at the configured device, as its own type rather than a raw `OSStatus` — a rejected pin used to log the same success line as an accepted one |
| `tools/audiotap/Sources/MicInputDevice.swift` | The input device the microphone diagnostics actually name — resolving the system default used to misreport a device the engine had been pointed at instead |
| `tools/audiotap/Sources/MicChannelMap.swift` | Decides when a discrete multi-channel mic input needs an explicit converter `channelMap` — `AVAudioConverter`'s implicit downmix silently writes digital silence for most non-stereo layouts |
| `tools/audiotap/Sources/MicConverterFactory.swift` | Builds the tap → WAV converter for one mic capture, applying `MicChannelMap`'s explicit channel selection when needed |
| `tools/audiotap/Sources/AudioCaptureSession.swift` | Orchestrator (start/stop, computes micDelay) |
| `tools/audiotap/Sources/AudioCaptureConfiguration.swift` | What one capture records and how — the single list a new capture option is added to |
| `tools/audiotap/Sources/AudioCaptureResult.swift` | Result struct |
| `tools/audiotap/Sources/LiveAudioBuffer.swift` | Real-time audio sample snapshot yielded from capture callbacks (CATap IOProc + AVAudioEngine input tap) |
| `tools/audiotap/Sources/CurrentLevel.swift` | Pure function: dBFS level read with staleness decay (stale tap → silence) |
| `tools/audiotap/Sources/LevelPublisher.swift` | Cross-thread dBFS slot: audio callback writes, UI thread reads |
| `tools/audiotap/Sources/AppNoiseFloorSensor.swift` | Rolling 120 s share of app-tap samples in buffers below one 16-bit step; raises a flag at 90%, clears at 50% (issue #699). Warning only: feeds `ChannelFaultMonitor`'s `noiseFloor` fault, never the shared energy definition or the tap itself |
| `tools/audiotap/Sources/ChannelSignalAges.swift` | How long ago a channel last delivered a buffer, and last delivered one carrying signal — a single dBFS reading can't tell "dead", "muted", and "quiet room" apart, ages can (feeds `ChannelFaultMonitor`) |
| `tools/audiotap/Sources/SilentTrackObserver.swift` | Notices when the app track enters and leaves a run of exact zeros while buffers keep arriving, so the transition is in the log and not only the end state (issue #672) |
| `tools/audiotap/Sources/SilentTrackDiagnostics.swift` | Owns the three things the silent-track instrumentation needs that a value type can't hold: a queue the HAL reads may safely block on, a guard so a non-returning read costs one parked thread instead of a growing pile, and the observer's state (issue #672) |
| `tools/audiotap/Sources/SilentTrackWatchdogPolicy.swift` | Decides when a run of exact zeros on the app track is worth rebuilding the tap for, how often, whether a rebuild helped, and when to stop trying (issue #672 part 2); also covers a tap that never delivered a first buffer at all (issue #693) |
| `tools/audiotap/Sources/SilentTrackWatchdogLimits.swift` | The silent-track watchdog's limits that user-facing copy names (issue #672), so help text and notifications interpolate the values the policy enforces instead of restating them |
| `tools/audiotap/Sources/DebugRMSReporter.swift` | Throttled RMS accumulator/reporter for audio debug logging |
| `tools/audiotap/Sources/Helpers.swift` | `machTicksToSeconds`, `getDefaultOutputDeviceUID`, `writeAllToFileHandle` |
| `tools/audiotap/Sources/MicRestartPolicy.swift` | Pure decision logic for mic engine restart on device change |
| `tools/audiotap/Sources/MicConfigurationChangePolicy.swift` | What a microphone engine's input unit is bound to and delivers: the device and the hardware-side format the tap was installed for |
| `tools/audiotap/Sources/DefaultInputListenerRegistrar.swift` | Registers and removes the mic handler's default-input-device-change listener behind a seam, so a test can watch add/remove without a real device change reaching it |
| `tools/audiotap/Sources/CaptureRestartRetryPolicy.swift` | Retry/backoff policy for a failed capture restart, shared by both the app-audio and mic channels (issue #379) |
| `tools/audiotap/Sources/RestartArbiter.swift` | Bounds how long a *single* restart attempt may run — generation-tagged so a wedged attempt that eventually returns is rejected rather than adopted, and forbids output-file creation forever once timed out (issue #588) |
| `tools/audiotap/Sources/OutputDeviceChangeCoordinator.swift` | State machine for output device change + tap restart flow |
| `tools/audiotap/Sources/ProcessTreeEnumerator.swift` | Enumerates all PIDs under an `.app` bundle (Electron/Teams child-process support) |
| `tools/audiotap/Sources/TappedProcess.swift` | One process the tap was built from, kept so its state can be read again later rather than re-translating a PID that may since have been reused by a different object |
| `tools/audiotap/Sources/ProcessResponsibility.swift` | Groups a helper process with the app macOS holds *responsible* for it — needed for Safari, whose call audio comes from WebKit XPC services outside `Safari.app` rather than child processes under its bundle (issue #524); private symbol via `dlsym`, so it is `nil` under `APPSTORE` |
| `tools/audiotap/Sources/ChromiumHelperFilter.swift` | Keeps a Chromium/Electron app's audio service (`--utility-sub-type=audio.mojom.AudioService`) in the tap set and drops its other helpers; tapping the video-capture service alongside stalled the tap before its first IO cycle (issue #791). Leaves the set alone when no audio service is found |
| `tools/audiotap/Sources/ProcessArguments.swift` | Reads another process's argv via `KERN_PROCARGS2` (pure parser + live sysctl reader); empty when the kernel will not say, which every caller reads as "no opinion" |
| `tools/audiotap/Sources/ProcessOutputState.swift` | What a tapped process's CoreAudio object says about its own output (issue #672); `kAudioProcessPropertyIsRunningOutput` reads 1 while the process is running IO, which is less useful than it looks |
| `tools/audiotap/Sources/SystemSettingsPaths.swift` | User-facing System Settings navigation paths (e.g. Screen Recording pane, renamed in macOS 15), kept in one place so the tap-error hint, permission UI, and channel-health notification name it identically |
| `tools/audiotap/Sources/SampleRateQuery.swift` | Pure functions for sample rate detection and cross-validation |
| `tools/audiotap/Sources/DeliveredRateTracker.swift` | Measures the rate the tap is actually delivering, so a device that renegotiates in place is followed (issue #673) — rate used to be measured once on the first IOProc callback and only re-measured after a restart |
| `tools/audiotap/Sources/AVAudioNode+SafeInstallTap.swift` | Safe `installTapOnBus` wrapper catching `NSException` via `CExceptionCatcher` (issue #379) |
| `tools/audiotap/Sources/AppAudioCapture+Resampling.swift` | Capture-time resampling for CATap buffers (line-cap split from `AppAudioCapture`) |
| `tools/audiotap/Sources/AppAudioCapture+TapError.swift` | Tap-creation error mapping (line-cap split from `AppAudioCapture`) |
| `tools/audiotap/Sources/CExceptionCatcher/` | Obj-C module catching AVFoundation `NSException` from `installTapOnBus` |
| `tools/audiotap/Sources/DebugTapFault.swift` | Fault-injection config for mic device-change E2E to verify NSException recovery |
| `tools/audiotap/Sources/MicCaptureHandler+Timeline.swift` | Timeline tracking for `MicCaptureHandler` across device-change restarts |
| `tools/audiotap/Sources/StreamingMonoResampler.swift` | Streaming mono resampler for the live 16 kHz audio path |
| `tools/audiotap/Sources/TapFormatResolver.swift` | Derives mic tap format from hardware format (prevents installTap channel-count mismatch) |
| `tools/audiotap/Sources/TimelineAnchor.swift` | Wall-clock timeline anchor across device-change restarts (keeps track aligned to real time) |

### Support

| File | Role |
|------|------|
| `TranscriberStatus.swift` | Status + state enum models |
| `AppPaths.swift` | Centralized path constants (ipcDir, dataDir, logSubsystem, speakersDB) |
| `GlobalHotkey.swift` | A key combination that works while another app is frontmost, as typed (e.g. "⌃⌥M") and stored since a key code alone cannot be displayed |
| `AXHelper.swift` | Shared accessibility API helper (used by ParticipantReader) |
| `NotificationManager.swift` | macOS notifications |
| `NotificationUrgency.swift` | How urgently a notification must reach the user — two cases (vs. the five `UNNotificationInterruptionLevel` offers) mapped to whether a given notification should pierce Focus |
| `NotificationScheduling.swift` | Port over the `UNUserNotificationCenter` slice `NotificationManager` uses, so posting/registration is testable against a fake scheduler |
| `NotificationRingBuffer.swift` | Bounded, thread-safe log of recently-posted notifications (`#if !APPSTORE`) |
| `SilentNotifier.swift` | No-op notifier for CLI targets and tests that don't need notifications |
| `DateFormatter+FilenameStamp.swift` | `DateFormatter` pinned to Gregorian calendar + POSIX locale for filename timestamp stamps |
| `KeychainHelper.swift` | Legacy keychain CRUD (token now file-based) |
| `BoundedProcess.swift` | A subprocess run with a time limit; `run(timeout:onStop:)` returns only once it has exited by itself, after being stopped at the limit, or after the calling task was cancelled — stops with SIGTERM then SIGKILL, both to the process group |
| `ProcessOutput.swift` | Reads what a subprocess writes to its pipes with no thread blocked in a read; each stream can also be ended from outside, at the latest a drain grace after the process exits, since a child it started can hold a pipe open past its own exit |
| `RecognitionStats.swift` | Recognition event model + `recognition_log.jsonl` reader/writer — backs `RecognitionStatsView` |
| `Permissions.swift` | Mic/accessibility permissions, project root detection |
| `PermissionRow.swift` | Permission status row UI component (icon, detail, help popover) |
| `PermissionHealthCheck.swift` | TCC verdict + live probe → `PermissionStatus`; drives exclamation badge overlay |
| `PermissionHealthCheck+MicProbeDeadline.swift` | The microphone permission probe under a deadline (issue #778): `probeMicrophone` brings up an `AVAudioEngine` through a call that can loop inside AVFAudio and never return (issue #588), so the probe itself is now bounded |
| `NotificationVisibility.swift` | What the notification centre will actually *do* with a posted notification (alert style, Time Sensitive, scheduled delivery) — authorisation alone can read `.authorized` while the prompt never shows |
| `ParticipantReader.swift` | Teams participant extraction via Accessibility API |
| `DebugRPCServer.swift` | Embedded HTTP RPC server core (routing, auth) for shell-driven inspection. `#if !APPSTORE`, opt-in via Settings → Advanced → Local Automation API or `MEETINGTRANSCRIBER_DEBUG_RPC=1`. Bearer-token + Origin reject; binds 127.0.0.1 only. Endpoint handlers split into companion files below (line-cap) |
| `DebugRPCServer+V1.swift` | `/v1` versioned automation API routing + response envelopes (`POST /v1/transcribe`, `/v1/jobs`, naming, `/v1/watch`, `/v1/record`) |
| `DebugRPCServer+Metrics.swift` | `GET /metrics` handler — cumulative CPU/RAM/instruction counters via `proc_pid_rusage` |
| `DebugRPCServer+Screenshot.swift` | `GET /screenshot` handler (PNG of the largest visible window via ScreenCaptureKit) |
| `DebugRPCServer+UITree.swift` | `GET /ui/tree` handler — read-only accessibility tree of an allowlisted window |
| `DebugRPCServer+UIPress.swift` | `POST /ui/press` handler — presses an allowlisted control via in-process `AXUIElementPerformAction`, or via `MouseInjection` when `"via":"click"` is requested |
| `DebugRPCServer+UIType.swift` | `POST /ui/type` handler — types into an allowlisted plain-text field via `KeyboardInjection` |
| `DebugRPCServer+AXElement.swift` | Shared self-pid `AXUIElement` tree walk backing `/ui/tree` and `/ui/press` |
| `DebugRPCServer+MeetingNotes.swift` | `POST /v1/jobs/<id>/notes/retry`, `POST /v1/jobs/notes/retry` and `POST /v1/jobs/notes/generate`, wired through `MeetingNotesActions` |
| `SpeakerDBActions.swift` | Outcome and closure types of the speaker-DB `/action/*` routes (line-cap split from `DebugRPCServer.swift`) |
| `KeyboardInjection.swift` | Types text into the app's own UI via posted key events (not an AX set-value, which wouldn't fire a SwiftUI `TextField` binding) — backs `POST /ui/type` |
| `MouseInjection.swift` | Clicks a point in the app's own UI via posted mouse events (real hit-testing, unlike some `kAXPressAction` controls that report success without effect) — backs `POST /ui/press` `"via":"click"` |
| `HTTPRequest.swift` | Minimal HTTP/1.1 request parsing for `DebugRPCServer` (pure value type, line-cap split) |
| `HTTPResponse.swift` | Minimal HTTP/1.1 response serialization for `DebugRPCServer` (pure value type, line-cap split) |
| `RPCResourceMetrics.swift` | JSON-serializable resource-usage snapshot of the running process, backs `GET /metrics` |
| `JobStatusDTO.swift` | Wire + persisted shape for `GET /v1/jobs/<id>` (live or terminal job status, result paths, and optional Notion page details) |
| `NamingStatusDTO.swift` | Wire shape for `GET /v1/jobs/<id>/naming` — per-speaker auto-name suggestion + speaking time + participants |
| `IdempotencyStore.swift` | Bounded FIFO map of `Idempotency-Key` → created job IDs for the `/v1` enqueue routes |
| `TerminalJobStore.swift` | File-backed store of recent finished-job statuses (survives queue reaping + app restart) |
| `AppSettings+RPC.swift` | Builds the read-only settings projection for the debug RPC `/state` endpoint |
| `AppState+RPC.swift` | Builds `RPCStateSnapshot` from live `AppState` for the `/state` endpoint (`#if !APPSTORE`) |
| `RPCStateSnapshot.swift` | JSON-serializable RPC state snapshot type (`#if !APPSTORE`) |
| `Bundle+AppVersion.swift` | Bundle extension: `appVersion` + `gitCommitHash` from `Info.plist` |
| `DiagnosticExporter.swift` | Reads log entries and writes shareable `.log` file (Settings → Advanced → Export Diagnostics) |
| `PersistentDiagnosticLog.swift` | Persistent `log stream` subprocess with sliding-window restart policy for long-term log retention |
| `DiagnosticsLogging.swift` | Where a component writes the diagnostics lines a field report is triaged from — a seam rather than a bare `Logger` so a test can assert that a line was written at the point it claims to be |
| `E2EMicFault.swift` | Which microphone fault the fault-injection build injects, chosen per launch; the whole type and the env var that selects a fault are compiled only with `-DE2E_FAULT_INJECTION` (`MTT_FAULT_INJECTION=1 scripts/run_app.sh`) |
| `String+LogRedaction.swift` | String extensions: `.pseudonymized` (SHA-256 4-hex prefix) and `.redactedName` for log privacy |
| `FileManager+OwnerOnly.swift` | `FileManager` extension: owner-only file permission constant (`rw-------`) as single source of truth |
| `SecurityScopeAccess.swift` | Opens and closes security-scoped access to a user-picked folder; access must be opened on the URL object that resolved from the security-scoped bookmark itself, never on a derived or path-rebuilt URL |
| `UserDefaults+Bool.swift` | A stored flag, or a fallback when the key was never written or holds something that isn't one — not `object(forKey:) as? Bool`, since an untyped `defaults write` stores a string that cast reads as missing |
| `SingleFlight.swift` | Single-flight async deduplication coordinator (concurrent callers await one shared run and receive its outcome) |
| `TerminationFlush.swift` | What AppKit waits for between a quit request and the process exit; `PipelineQueue.saveSnapshot()` hands its write to a detached worker, so a quit inside that gap would otherwise exit with the snapshot one state transition behind |
| `RPCServerController.swift` | `@Observable` controller owning `DebugRPCServer` lifecycle (`#if !APPSTORE`, wired by `AppState`) |
| `PermissionsController.swift` | `@Observable` controller for permission health checks (wired by `AppState`, re-runs on activation) |

### Companion CLIs

| Path | Role |
|------|------|
| `tools/mt-cli/` | Thin Swift client for `DebugRPCServer`. Subcommands: `state`, `healthz`, `screenshot`, `open-settings`, `close-settings`, `confirm-browser-consent`, `wav-verdict`, `seed-speaker`, `rename-speaker`, `delete-speaker`, `merge-speakers`, `ui-tree`, `ui-press`, `watch`, `record`. Reads token from `~/Library/Application Support/MeetingTranscriber/.rpc-token`. Skill doc at `tools/mt-cli/skill.md`. |
| `tools/meeting-simulator/` | Test fixture: spawns a fake meeting window for E2E detection tests |

---

## State Machine

```
WatchLoop:     idle → watching → recording → watching (enqueues PipelineJob)
PipelineQueue: waiting → transcribing → [diarizing] → generatingProtocol → done (Notion export runs in the background; 60s auto-remove after any pending export, deferred while a native menu is open)
                                                                            ↳ error
```

**Transitions** are observable via `WatchLoop.state` and `PipelineQueue.jobs`, triggering:
- Menu bar icon/label updates
- macOS notifications (recording started, protocol ready, error)

### Menu Bar Icon Animations

`BadgeKind.compute(watchLoopActive:watchLoopState:transcriberState:activeJobState:updateAvailable:permissionProblem:consentPending:)` is the pure function that maps the combined `WatchLoop` + pipeline state into one of `BadgeKind.inactive | .recording | .transcribing | .diarizing | .processing | .userAction | .done | .error | .updateAvailable`. `MenuBarIcon.image(badge:permissionOverlay:recordOnlyOverlay:)` then renders the matching animation frame.

| State | GIF | Triggered by | Code path |
|-------|-----|--------------|-----------|
| **Idle** | <img src="menu-bar-idle.gif" width="60"> | `WatchLoop.state == .idle / .watching` and `PipelineQueue` empty | `BadgeKind.inactive` |
| **Recording** | <img src="menu-bar-recording.gif" width="60"> | `WatchLoop.state == .recording` (waveform bars bounce) | `BadgeKind.recording` |
| **Transcribing** | <img src="menu-bar-transcribing.gif" width="60"> | `PipelineJob.state == .transcribing` or `TranscriberState.recordingDone` (bars morph into text glyphs) | `BadgeKind.transcribing` |
| **Diarizing** | <img src="menu-bar-diarizing.gif" width="60"> | `PipelineJob.state == .diarizing` (bars split into colored speaker groups) | `BadgeKind.diarizing` |
| **Protocol** | <img src="menu-bar-protocol.gif" width="60"> | `PipelineJob.state == .generatingProtocol` (lines appear sequentially) | `BadgeKind.processing` |

The icon is rendered as a SwiftUI `Image` template (auto-tinted by AppKit for light/dark mode) **unless** an overlay applies — overlays force non-template rendering to keep the colored badge intact.

### Permission problem badge

<p>
<img src="menu-bar-permission.gif" width="80" alt="Permission problem badge">
</p>

A red circle with a white "!" is composited in the bottom-right corner by `MenuBarIcon.drawExclamationBadge` whenever `PermissionHealthCheck` reports any of Microphone / Screen Recording / Accessibility as `.denied` or `.broken`. The overlay sits **on top of whatever primary state animation** is currently active — the user still sees what the app is doing while being told something is wrong. See "Permission health check + badge overlay" below for the full health-check semantics.

### Record-only mode badge

<p>
<img src="menu-bar-record-only.gif" width="80" alt="Record-only mode">
</p>

A persistent small red dot in the bottom-right corner indicates that **Record-only mode** is enabled (`AppSettings.recordOnly == true`). In this mode `WatchLoop.enqueueRecording()` moves dual-source WAVs into `<outputDir>/recordings/` together with a `<basename>_meta.json` `RecordingSidecar` and skips the entire post-processing pipeline (VAD, transcription, diarization, protocol). Intended for fleet topologies where macOS clients capture and a separate machine (e.g. a Linux GPU host via Syncthing) processes the audio. The sidecar's `trigger` field (`auto` | `manual`, schema version 2) tells that consumer which call site produced the recording, so it can treat a short auto-detected capture (likely a false trigger) differently from a short deliberate manual one.

Like the permission badge, the dot is rendered as a persistent overlay on top of whatever primary animation is currently active — so the mode is always clearly indicated whether the app is idle, recording, or running anything else. **Precedence:** when both apply, the red exclamation (permission badge) wins, because a permission problem actually breaks recording while record-only is a deliberate user choice.

### Per-channel asymmetric-silence indicator

When one capture channel goes silent while the other is still carrying audio for longer than the configured debounce window, the waveform bars in the menu bar are tinted **red** to surface the half-broken capture at a glance. `MenuBarIcon.image(..., micSilentOverlay:appSilentOverlay:)` paints the **top half** red when the mic channel is the silent one and the **bottom half** red when the app-audio channel is the silent one. When both apply, both halves are red (effectively all-red bars). Like the permission badge, this overlay forces non-template rendering so the red stays red in dark mode.

The flags driving this overlay (`AppState.micSilentActive` / `AppState.appSilentActive`) are flipped by a ~10 Hz polling task that reads `WatchLoop.activeRecorder?.{mic,app}LevelDBFS` and feeds the values into a pure `ChannelHealthMonitor` state machine. The monitor uses two dBFS thresholds — `silenceThresholdDBFS` (-60) and `speechThresholdDBFS` (-50) — with hysteresis: an episode only starts when one channel is below silence *and* the other is above speech, and only resolves when the supposedly-silent side crosses back above the speech threshold. Transient dips into the dead zone between the thresholds (natural pauses between syllables) keep the debounce timer running rather than resetting it.

**Settings → Audio → Per-Channel Indicator** turns the tint on or off (default on); the **Warn after** slider (30–300 s, default 90 s) sets the window. The toggle governs the tint alone: notifications are posted whether or not it is on, because a user who finds the red icon distracting is not asking to be left uninformed about a dead capture channel. The per-channel notification comes from `ChannelFaultMonitor`, which asks whether that channel has stopped delivering (no buffers, or buffers of digital zeroes) rather than whether it is quiet. On the app channel it also reports a floor below one 16-bit step (`AppNoiseFloorSensor`, issue #699): buffers that carry energy, so they never read as digital silence, but for two minutes at least 90% of them sit below 2^-15 RMS, which a tap of a silent meeting app never delivers. The symmetric `Recording Appears Silent` still comes from `SilentRecordingMonitor` and its dBFS thresholds, so it remains level-based and can fire on a call in which nobody is speaking; moving it onto the ages needs the gap between the two monitors closed first.

**Precedence ordering** (highest wins, composes over the others underneath):

1. Permission badge (red exclamation) — actually breaks recording
2. Channel-silent tint (red waveform halves) — degraded recording
3. Record-only dot (persistent red dot) — user-chosen mode
4. Primary state animation (idle / recording / transcribing / diarizing / protocol)

---

## Audio Pipeline

### Capture

```
AudioTapLib (CATapDescription)
├─ Input: App PID → CoreAudio process tap → aggregate device
├─ Output: Interleaved float32 (mono or stereo) → FileHandle (raw PCM)
├─ Mic: AVAudioEngine → mono WAV file (MicCaptureHandler)
└─ Metadata: micDelay, actualSampleRate, actualChannels via AudioCaptureResult
```

**Key:** CATapDescription is TCC-gated: it needs the "Audio Recording" grant (`NSAudioCaptureUsageDescription`) or, as a fallback, Screen Recording. With neither, the tap starts without an error and captures silence. Handles output device changes by recreating tap automatically.

**Start order (issue #693):** `AudioCaptureSession.start()` opens the microphone first and the app tap second, and the order is load-bearing. Opening the input takes a Bluetooth headset out of A2DP into its call profile; with the tap opened first, the disturbance landed underneath an aggregate device that had been created and started but had not yet run its first IO cycle, and the tap then delivered nothing for the rest of the recording (no IO callback, zero bytes, no error). Measured with a throwaway probe that is not in this repository: 9 failures in 30 starts with the tap first, 0 in 30 with the microphone first, which bounds what is left rather than proving it gone. **The full argument, including what the order does and does not guarantee and the three accepted costs, is the doc comment on `AudioCaptureSession.start()`; it is kept in one place on purpose.** Downstream, the measured `micDelay` is now normally negative (the microphone leads), which `MicDelayNormalisation` turns into a padded app track and a reported delay of 0; two cases it does not cover are a dual-source recording whose `_mic.wav` turns out unreadable, where the raw negative value is still what gets reported, and a crash-recovered recording, which `DualSourceRecorder.recoverCrashedRecording` rebuilds with a hard-coded `micDelay: 0` and therefore does not realign at all.

**Restart bounding (issue #588):** a device-change restart on either channel can wedge inside AVFAudio/CoreAudio and never return (e.g. `AVAudioEngine.inputNode` looping on a dangling Bluetooth sub-device held by coreaudiod). `RestartArbiter` bounds how long a single restart attempt may run — attempts carry a generation and run off the main queue, so a result from a wedged attempt that eventually returns is rejected rather than adopted. `CaptureRestartRetryPolicy` bounds how many attempts are made and is shared by both channels. A channel that gives up tells the user directly (`AudioCaptureSession`/`DualSourceRecorder` expose which one) instead of only decaying to silence, which the asymmetric-silence detector would otherwise misreport as a routing/mute problem rather than a channel that is gone for good.

**Safari support (issue #524):** the app-audio tap targets processes via macOS's *responsible-process* attribution (`ProcessResponsibility`) as well as bundle-path enumeration (`ProcessTreeEnumerator`) — Safari's call audio comes from WebKit XPC services outside `Safari.app`, unlike Electron/Chrome helpers that live inside their own bundle.

### Processing (DualSourceRecorder.stop())

```
App temp: already 16kHz mono float32 (resampled in-IOProc at capture time)
  → Save app.wav (16kHz mono)
  → Load mic.wav (already 16kHz from MicCaptureHandler)
  → Echo suppression (RMS-based gate, 20ms windows)
  → Delay alignment (prepend zeros by MIC_DELAY)
  → Mix (average tracks)
  → Save mix.wav (16kHz mono)
```

All recordings are normalized to 16kHz at capture time — no resampling needed in the pipeline.

### Echo Mitigation (issue #581)

The RMS-based echo suppression above is a capture-time gate; it does not catch a loudspeaker recording where the remote voices bleed into the microphone track after both tracks are already saved. Two independent, non-composing remedies run in the transcribe stage, decided per recording by `EchoRemedy`:

- **Detection first, always:** `EchoBleedDetector` (`PipelineQueue+EchoBleed.swift`) scores per-10s-window envelope correlation between the app and mic tracks, centred on the recorder's own `micDelay`, and reports `.notMeasured` / `.clean` / `.affected` as `PipelineJob.echo` (surfaced on `GET /v1/jobs/<id>`).
- **Cancellation (off by default):** `LocalVQECanceller`, over the vendored `CLocalVQE` static library, removes the far end from the microphone *audio* before transcription. `EchoCancellationSelfCheck` judges the run from the canceller's own per-window report before its output is adopted.
- **Transcript dedup (opt-in, off by default, and only on an `.affected` verdict):** `EchoSegmentClassifier` decides per mic segment whether the app track's energy explains it; explained segments are marked `suppressed` and left out of the transcript, so the far end isn't written twice. It ships off because a local speaker who only ever talks while the far end does is explained away and removed with them, which `EchoSegmentClassifierTests` pins as an expected failure.
- **Embedding quarantine:** on an `.affected` recording, `EchoEmbeddingQuarantine` holds mic-track speaker embeddings back from `speakers.json`, since a voice learned from contaminated audio would otherwise be folded permanently into a stored centroid.

Full design rationale (why the two remedies can't compose, the model provisioning, and the self-check semantics) lives in `docs/architecture/echo.md`.

---

## Transcription

### Engine Selection

`TranscribingEngine` protocol abstracts ASR backends. `AppSettings.transcriptionEngine` selects the active engine.

| | WhisperKit | Parakeet TDT v3 |
|---|---|---|
| **Languages** | 99+ | 25 European |
| **Model size** | ~800 MB–1.5 GB | ~50 MB |
| **Speed (M4 Pro)** | ~10–20× RTF | ~110× RTF |
| **Language selection** | Manual or auto-detect | Auto-detect only |
| **Timestamps** | Per-segment | Per-token |
| **macOS** | 14+ | 14+ |
| **Hallucinations** | Can occur | Minimal |

### WhisperKit Engine

- **Model:** `openai_whisper-large-v3-v20240930_turbo` (CoreML/ANE)
- **Pre-loading:** Model downloaded and loaded at app launch (when selected)
- **Lazy fallback:** `ensureModel()` loads on-demand if not ready

### Parakeet Engine

- **Model:** NVIDIA Parakeet TDT v3 via FluidAudio (CoreML/ANE)
- **Pre-loading:** Model downloaded and loaded at app launch (when selected)
- **Token grouping:** `groupTokensIntoSegments` groups per-token timings into sentence-level segments (split on `. ! ?` or 20 tokens)

### Modes

1. **Single source:** `transcribeSegments(audioPath:)` → `[TimestampedSegment]` with start/end/text
2. **Dual source:** `transcribeSegments(appAudio:)` + `transcribeSegments(micAudio:)` → `mergeDualSourceSegments(appSegments:micSegments:)` → `[TimestampedSegment]` merged by timestamp
   - App segments labeled "Remote"
   - Mic segments labeled with user's mic name (default "Me")
   - `mergeDualSourceSegments` is a protocol extension on `TranscribingEngine` — shared by all engines

### Post-processing (WhisperKit only)

- **Token stripping:** Regex `<\|[^|]*\|>` removes `<|startoftranscript|>`, `<|en|>`, etc.
- **Hallucination filtering:** Skip consecutive identical segments

### Custom Vocabulary & Terminology Normalization

- **Custom vocabulary** is one shared one-term-per-line file (`AppSettings.customVocabularyPath`, resolved through `VocabularyFileAccess` for sandbox bookmark access), not per engine. `AppSettings.customVocabularyEnabled` switches it on or off for both engines. Parakeet applies it via CTC boosting. WhisperKit applies it via `WhisperVocabularyPrompt` turning the file into a bounded decoder-prompt hint — WhisperKit shares its 224-token decoder context between that prompt and generated tokens, so a dense recording can lose whole sentences to it; Parakeet is the recommended engine for vocabulary boosting.
- **Terminology normalization** (`TerminologyNormalizer`) is a separate, engine-independent post-ASR pass, applied to transcript segments right after transcription: opt-in `Canonical => spoken variant | another variant` rules, configured in Settings → Transcription → Vocabulary → Canonical terminology, either in the text area or in an optional rules file that is read again at the start of every job (details in `docs/architecture/transcription.md`).

---

## Live Captions (PoC)

Optional in-meeting caption overlay, "Enable live transcription during recording" in Settings →
Transcription (`AppSettings.liveTranscriptionEnabled`, off by default; enabling downloads a ~0.6 GB
model on first use behind a consent alert). Nested "Show caption overlay"
(`AppSettings.liveCaptionsOverlayEnabled`, default on) hides `LiveCaptionsOverlay` without
stopping the pipeline; `LiveCaptionsState` and `/state.liveCaptions` keep updating.

`LiveCaptionsGate.strategy(liveEnabled:engineLanguage:engineSupportsLive:)` is the pure decision
function — shared by `AppState`, `LiveTranscriptionCoordinator`, and `LiveTranscriptionController`
— that picks the per-channel backend from the active engine's **explicitly configured** language
(not auto-detect):

| Engine language | Strategy | Backend |
|---|---|---|
| `en` | English streaming | `EouStreamingCaptionSession` (FluidAudio Parakeet EOU) |
| a language Nemotron was trained on (`LiveCaptionLanguages`) | Nemotron streaming | `NemotronStreamingCaptionSession` / `NemotronAsrManager` (FluidAudio Nemotron multilingual) |
| auto-detect (`nil`) or any other language | re-transcribe | `StreamingTranscriber` via the active engine's `transcribeSamples`, if supported |
| auto-detect or any other language + engine doesn't support live re-transcribe | none | captions off |

Both streaming sessions are engine-independent (they drive their own FluidAudio models directly),
so they're available even when the active `TranscribingEngine` has no live re-transcribe hook.
`LiveTranscriptionController` wires the resolved per-channel pipeline to both `DualSourceRecorder`
sinks (mic + app) and feeds `LiveCaptionsState`, which backs the `LiveCaptionsOverlay` window.
`ModelWarmupQueue` serializes model warm-up loads (ASR engine + streaming models) so they don't
compile/load concurrently at launch.

---

## Diarization

### FluidDiarizer (On-device)

On-device speaker diarization using FluidAudio (CoreML/ANE). No HuggingFace token or Python subprocess needed. Models downloaded automatically on first run (~50 MB); Nemotron 3 mode downloads its own ~200 MB model the first time it is used.

Three modes selected via `AppSettings.diarizerMode`:
- **`.offline`** (default) — `OfflineDiarizerManager`, standard speaker segmentation
- **`.sortformer`** — `SortformerDiarizer`, overlap-aware diarization (handles simultaneous speech); speaker embeddings extracted post-hoc via `FluidDiarizer+SortformerEmbeddings` using overlap-excluded WeSpeaker masks
- **`.nemotron`** — `Nemotron3Diarizer`, NVIDIA's Nemotron 3 Diarization (an 8-speaker streaming Sortformer), overlap-aware for up to eight speakers; the `fast128` preset (~200 MB, downloaded on first use). The recording is streamed through the model in 60 s pieces, so the mel spectrogram of the whole file is never held; embeddings come from the same post-hoc WeSpeaker extraction as `.sortformer`. Like `.sortformer`, it ignores `numSpeakers`

Flow: `FluidDiarizer.run(audioPath, numSpeakers)` → selected diarizer → `DiarizationResult` with segments, speaking times, and speaker embeddings.

**Experimental tuning:** Five `OfflineDiarizerConfig` parameters are exposed in `AppSettings` and editable in Settings → Speakers → Experimental: Diarization Tuning: `clusterThreshold` (0.6), `warmStartFa` (0.07), `warmStartFb` (0.8), `minSegmentDurationSeconds` (1.0), `excludeOverlap` (true). All default to FluidAudio community values.

### Speaker Matching

`SpeakerMatcher` matches diarization speaker embeddings against a persistent speaker database (`speakers.json`) using cosine similarity (threshold: 0.40, confidence margin: 0.10). Stores a running-mean centroid (primary match anchor) plus a recent-samples FIFO (max 3, fallback when centroid match is borderline). Quality filter: embeddings from segments shorter than 3 s are excluded from the centroid but kept as fallback samples. Speakers are ranked by recency and use count in the naming UI. Enables recognition of returning speakers across meetings.

### Speaker Assignment

For each transcript segment, find the diarization segment with the longest temporal overlap:
```
overlap = max(0, min(seg.end, dSeg.end) - max(seg.start, dSeg.start))
```
No overlap → nearest-segment fallback by gap distance. Only if no diarization segments exist → "UNKNOWN".

### Dual-Track Diarization

When dual-source recording (app + mic) is available:
1. Transcribe app/mic tracks separately → "Remote" / micLabel segments
2. Diarize app track and mic track separately via FluidAudio
3. `mergeDualTrackDiarization()` — prefix speaker IDs (`R_` for remote, `M_` for local), merge segments by time
4. `preMatchParticipants()` — heuristic assignment of Teams participants to unmatched speakers by speaking time
5. Speaker naming UI — all speakers editable with participant suggestions
6. `assignSpeakersDualTrack()` — app segments matched against app diarization, mic segments against mic diarization

**Single-source fallback:** When only mix audio is available, diarize the mix and use `assignSpeakers()` with nearest-segment fallback.

### Speaker Naming UI

`SpeakerNamingView` shown when diarization finds speakers: by itself while "Open naming window automatically" (`AppSettings.autoOpenSpeakerNaming`) is on, otherwise when the user opens it from "Name Speakers…" in the menu bar. Each row shows label, auto-matched name, speaking time, and audio playback. All rows are editable. Supports re-run with different speaker count.

---

## Protocol Generation

### Provider Selection

`AppSettings.protocolProvider` selects the active provider:
- **`.claudeCLI`** — Claude CLI subprocess (`#if !APPSTORE`)
- **`.openAICompatible`** — Any OpenAI-compatible HTTP API (Ollama, LM Studio, llama.cpp, etc.)
- **`.none`** — Skip LLM generation; save transcript only

`AppSettings.includeFullTranscriptInProtocol` and
`AppSettings.saveRawTranscriptSeparately` default to `true` to preserve the
legacy output. Their values are captured when a job enters the queue, so a
later settings change affects only subsequent recordings. The raw `.txt` and
verbatim segment sidecar are deleted only after a job completes successfully
with a saved protocol. If protocol generation is disabled, fails, or the job
ends in an error, the raw transcript is retained with a warning; recordings are
governed by their separate retention policy.

`AppSettings.useAITitleForFilenames` (off by default, captured per job the same
way) names a job's files after the meeting title in the generated protocol's
heading instead of the timestamped slug: `sprint_planning.md`,
`sprint_planning.txt`, `sprint_planning_mix.wav` and so on. It can only act once
a protocol exists, so with the `.none` provider, a failed generation, or a
heading that is a placeholder, bracketed or has no alphanumerics, the
timestamped names stay. The rename moves every artifact of the job in both
`protocols/` and `recordings/`, found from the closed suffix lists rather than a
prefix scan, because a bare-title stem is a prefix of other jobs' stems. A title
is taken only if none of its artifacts exist, otherwise `_<shortID>` is
appended, and nothing is overwritten. If any move fails, the ones already done
are undone and the job keeps its old stem and paths.

`AppSettings.protocolLanguage` (default `"German"`) is substituted into the prompt as `{LANGUAGE}`. Custom prompts can also use `{MEETING_DATE}` (`YYYY-MM-DD`) and `{MEETING_TIME}` (`HH:mm`), derived from the captured recording start time. Imports and recovery jobs resolve those time placeholders to `Unknown` rather than their enqueue or processing time. Only recordings with a captured start receive the authoritative meeting-metadata block, so existing custom prompts receive reliable temporal context without needing to add the placeholders.

### Notion Export

When `AppSettings.notionExportEnabled` is on, `PipelineQueue` saves the generated
protocol locally first, moves the job to `done`, then exports the Markdown to the
selected Notion database in the background, so a slow Notion cannot hold up the next
job and a quit costs no second LLM call. The local copy is kept by default;
`notionSkipLocalMarkdownOnSuccess` removes it only after a successful export or
retry. Export failures add a job warning with a short cause and keep the local copy.
`notionSpeakerTagsEnabled` separately opts in to sending confirmed speaker names as
tags and defaults off. Date and Tags are written only when the selected database
has recognized columns.

The job records its Notion page and database IDs as soon as the page is created,
so retries and protocol re-exports can use the same destination. Re-export after
speaker renaming replaces the page's root content with the current protocol. New
content is appended before prior root blocks are archived: a failed append keeps
the previous content, and a retry reconciles any partial new content. A successful
re-export replaces manual edits to the app-owned page, except blocks of a type
that never holds protocol text (`NotionReplacePlan.keptTypes`: sub-pages,
databases, media, embeds, links), which stay, above the new content. A delete
that answers 404, or 400 for an archived block, counts as done once a fresh
listing no longer shows the block; Notion answers 404 for lost access as well.
The client pins Notion API version `2022-06-28`; databases with multiple data
sources are rejected rather than guessing which source should receive the meeting.

A failed export retains the completed job and its local Markdown backup across
queue cleanup and app restarts, even if the original audio is no longer present.
The menu's **Retry Notion Export** action is available when export is enabled and
the backup file exists; it reads the saved protocol and reuses the recorded page
ID for the same database, plus saved speaker names when tags are enabled. It does
not transcribe audio or generate a new protocol. Success clears the export-failure
warning and starts the normal 60-second completed-job lifetime. If a completed
job's in-progress export is interrupted, recovery marks it failed with an
"interrupted when the app quit" warning for this same retry path; see
`pipeline.md` for when these jobs come back at launch. If the app stops during protocol generation for a pending export,
recovery regenerates from the saved corrected transcript when the saved
transcript and naming state permit protocol-only recovery, even if an older
Markdown file exists.

When a Notion target is routed and the Notion desktop app is running, the app
receives the page handoff. Otherwise, the original Notion URL is opened with
the system's default handler. If the desktop app rejects the handoff, the
original URL is also opened with the default handler. Local protocol files
always use the default handler.

### Claude CLI Invocation

```bash
/usr/bin/env claude -p - --output-format stream-json --verbose --model sonnet --tools "" --strict-mcp-config
```

- **Input:** Meeting metadata + protocol prompt (with variables substituted) + transcript piped to stdin
- **No tools:** `--tools ""` and `--strict-mcp-config` start the session without built-in tools or MCP servers, since the transcript is untrusted text. A CLI too old for these options fails the protocol with a message asking for an update; it is never run without them
- **Output:** Stream-json parsed line-by-line (content_block_delta + assistant message)
- **Environment:** `CLAUDECODE` env var stripped to allow nested invocation
- **Timeout:** 10 minutes

### Output Structure

```markdown
# Meeting Protocol - [Title]
## Summary
## Participants
## Topics Discussed
## Decisions
## Tasks (table)
## Open Questions

---

## Full Transcript
[appended when enabled]
```

---

## Data Flow

### Observable State Propagation

```
AppSettings (UserDefaults)
  → WatchLoop (@Observable: state, detail, currentMeeting, lastError)
  → PipelineQueue (@Observable: jobs, isProcessing, pendingSpeakerNaming)
    → AppState (computed: currentBadge via BadgeKind.compute(), currentStatus, currentStateLabel)
      → MeetingTranscriberApp (reads appState.*, passes to views)
        → MenuBarView (receives status + callbacks + pipeline queue)
        → SettingsView (receives @Bindable settings)
        → SpeakerNamingView (receives pendingSpeakerNaming data)
```

### File Locations

| Content | Path |
|---------|------|
| Recordings | `~/Library/Application Support/MeetingTranscriber/recordings/` |
| Protocols | `~/Downloads/MeetingTranscriber/` by default, or the folder chosen in Settings → Output |
| IPC | `~/Library/Application Support/MeetingTranscriber/ipc/` (`~/.meeting-transcriber/` is only read once, as a migration source) |
| Speaker DB | `~/Library/Application Support/MeetingTranscriber/speakers.json` |
| WhisperKit models | `~/Library/Application Support/MeetingTranscriber/huggingface/` |
| Pipeline logs | `~/Library/Application Support/MeetingTranscriber/ipc/pipeline_queue.json`, `pipeline_log.jsonl` |
| AudioTapLib | Linked as SPM library (no separate binary) |

---

## Testing Hooks

| Component | Injection Point |
|-----------|----------------|
| MeetingDetector | `windowListProvider` closure (mock window list) |
| PowerAssertionDetector | `assertionProvider` + `windowListProvider` closures |
| MicInputDetector | `processProvider` + `windowListProvider` + `responsibleAppProvider` closures |
| DiarizationProvider | `diarizationFactory` closure in PipelineQueue |
| ProtocolGenerating | `protocolGenerator` protocol in PipelineQueue |
| RecordingProvider | `recorderFactory` closure in WatchLoop |
| ProtocolGenerator | `claudeBin` parameter |
| NotificationManager | `NotificationScheduling` port over `UNUserNotificationCenter` (fake scheduler in tests) |
| ConsentPromptCoordinator | Injected timeout clock; pure register/resolve-once API (no UI dependency) |
| LiveCaptionsGate.strategy | Pure static function — call directly with any input combination, no controller needed |
| AppNotifying | `notifier` parameter in `AppState.init` (`SilentNotifier` default, `RecordingNotifier` in tests) |
| BadgeKind.compute | Pure static function — call directly with any input combination, no WatchLoop needed |
| DebugRPCServer | Out-of-process inspection via HTTP. Debug endpoints: `GET /state /healthz /metrics /screenshot`, `POST /action/openSettings /action/closeSettings`. Versioned automation API under `/v1` (`POST /v1/transcribe`, `POST /v1/jobs`, `GET /v1/jobs/<id>`, `GET`/`POST /v1/jobs/<id>/naming`, `POST /v1/jobs/<id>/naming/skip`, `POST /v1/jobs/<id>/notes/retry`, `POST /v1/jobs/notes/retry`, `POST /v1/jobs/notes/generate`, `GET`/`POST /v1/watch`, `GET`/`POST /v1/record`); see `docs/automation-api.md`. `#if !APPSTORE`; started by the Settings toggle or the env flag. `boundPort` exposes OS-assigned port for in-process integration tests. `tools/mt-cli/` is the matching inspection CLI. `scripts/test_rpc.sh` is a live smoketest (build + launch + drive + assert). |

---

## Permissions

| Permission | Required For | Notes |
|------------|-------------|-------|
| Screen Recording | Real meeting titles; fallback grant for the app-audio tap. Detection does not need it | CGWindowListCopyWindowInfo |
| Microphone | Mic recording | AVAudioEngine |
| Accessibility | Participant reading (optional; not applicable in the App Store build) | Teams AX tree |
| Audio Recording | App audio capture | CATapDescription (Screen Recording also accepted) |

### Permission health check + badge overlay

`PermissionHealthCheck` verifies each of the three TCC permissions the build uses by combining the system verdict with a live probe (e.g. `CGWindowListCopyWindowInfo` returning non-empty window titles for Screen Recording). Each permission resolves to `PermissionStatus.healthy | .denied | .broken | .notDetermined | .notApplicable` (the last is the App Store build's Accessibility verdict: that build does not use the grant, so it is not probed and raises no badge or notification) — `.broken` means "TCC says allowed but the probe disagrees," which happens when macOS hasn't actually wired the permission through and the user needs to toggle it off and on in System Settings.

`WatchLoop` runs the check on startup and `AppState` re-runs it on app activation. When the result is unhealthy:

1. `MeetingTranscriberApp` passes `permissionOverlay: true` to `MenuBarIcon.image(...)`, which composites a red circle with a white "!" in the bottom-right corner over the current badge (`MenuBarIcon.drawExclamationBadge`). This bypasses the cached template icons and renders a non-template image because the overlay must stay red in both light and dark mode.
2. `BadgeKind.compute(...)` returns `.error` when idle-with-problem, so the icon also reflects the problem state when no job is active.
3. A notification is posted via `NotificationManager` with the list of affected permissions (deduplicated — only re-posted when the problem set actually changes).

The overlay lives over the *currently active* animation (idle, recording, transcribing, …) so the user still sees what the app is doing and is simultaneously told "one of the permissions is wrong."

<p>
<img src="menu-bar-permission.gif" width="80" alt="Permission problem badge">
</p>

---

## Settings UI

`SettingsView` is a thin `NavigationSplitView` shell: a sidebar lists the tabs, the detail pane shows the selected one. Each tab is a self-contained `View` in `Sources/Settings/` owning only the bindings it needs and its own local `@State`. The settings window is resizable (`minWidth: 760, idealWidth: 860, maxWidth: 1000`).

| Tab | Sections | Bindings | Local state |
|---|---|---|---|
| **General** | Startup · Mode · Apps to Watch · Detection | `settings` | — |
| **Audio** | Microphone · VAD · Echo · Per-Channel Indicator | `settings` | `audioDevices` |
| **Transcription** | Engine + per-engine options + status + idle-model retention | `settings`, three engines | — |
| **Speakers** | Diarization · Speaker Identity · Known Voices · Recognition Stats · Experimental: Diarization Tuning | `settings`, `recognitionStatsLog`, `enrollmentDiarizerFactory` | `knownVoicesSheet` |
| **Output** | LLM Provider · Notes Language · Output Folder · Recording Retention · Prompt | `settings` | `claudeBinaries` (#if !APPSTORE), connection-test state, `availableModels`, `hasCustomPrompt` |
| **Advanced** | Permissions · Diagnostics | — | `micPermission`, `screenRecordingOK`, `accessibilityOK` |
| **About** | Build identity · Dependencies · Updates | `settings`, `updateChecker?` | — |

**Conditional rendering rules:**
- `noMic` hides the mic-device picker (Audio) and the Speaker Identity section (Speakers)
- `diarize` hides the diarizer-mode picker and Expected Speakers stepper
- `vadEnabled` hides the VAD threshold slider
- `transcriptionEngine` switches between WhisperKit / Parakeet option panels
- `unloadModelsWhenIdle` (default on) skips the launch ASR load (a missing model is still downloaded at launch) and releases in-memory ASR and batch VAD models at a safe pipeline drain; the toggle sits beside model status and "Load Model". Turning it off restores launch preload and retains models across drains and engine switches. With the setting on, unloaded status reads "Loads when needed" while `/state` continues to report `"unloaded"`.
- `protocolProvider` switches between Claude CLI / OpenAI-compatible / None panels; the OpenAI-compatible panel starts with the saved-connection controls (`OpenAIConnectionControls`, see `docs/architecture/pipeline.md`)
- `#if APPSTORE` removes the Claude CLI provider option entirely
- `updateChecker == nil` hides the entire Updates section in About

**Cross-cutting concerns owned by sub-views:**
- `OutputSettingsView` owns OpenAI-endpoint connection testing (`testConnection()`) and custom-prompt I/O (`openCustomPrompt`, `importCustomPrompt`, reset confirmation)
- `AdvancedSettingsView` owns permission live-probing (`refreshPermissions()`)
- `AboutSettingsView` owns version/build/ffmpeg status and update-check controls

---

## Key Architectural Decisions

1. **@Observable over @StateObject** — Fine-grained reactivity, macOS 14+
2. **PipelineQueue decoupling** — Recording and post-processing run independently; WatchLoop enqueues jobs and resumes watching
3. **AudioTapLib as SPM library** — Direct in-process audio capture via CATapDescription (App Store compatible)
4. **Dual-source recording** — Enables speaker separation without diarization (app=Remote, mic=Me)
5. **Graceful degradation** — Diarization optional, continues on partial failure
6. **Pre-loaded model** — Selected engine (WhisperKit or Parakeet) loaded at app launch, prevents delay on first meeting
7. **5s cooldown** — Prevents re-detecting same meeting after handling
8. **FluidAudio on-device diarization** — Replaces Python pyannote subprocess, no external dependencies
9. **Dual-track diarization** — App and mic tracks diarized separately, avoiding echo/cross-talk interference
10. **Embedded debug RPC + automation API** — In-process HTTP server (`DebugRPCServer`) exposes state, resource metrics (`GET /metrics`), screenshot, and scene actions for shell-driven inspection and integration tests, plus a versioned `/v1` automation API (headless transcribe + job/naming control, plus idempotent `/v1/watch` and `/v1/record` control surfaces behind Stream Deck/hotkey control; reference in `docs/automation-api.md`). Off by default, opt-in via the `Settings → Advanced → Local Automation API` toggle or the `MEETINGTRANSCRIBER_DEBUG_RPC=1` env var, excluded from App Store builds via `#if !APPSTORE`. Action endpoints route through existing `Notification.Name` observers in `MeetingTranscriberApp`, so RPC-driven flows mirror real menu-bar paths.
11. **No expensive work in SwiftUI hot paths** — view bodies, computed properties read by the body, and per-render closures must not call disk I/O, JSON decode, factory constructors, regex compilation, or other non-trivial work. SwiftUI re-renders on every `@State`/`@Observable` change and fans out aggressively, so what looks cheap once becomes a CPU pin fast. Push heavy values up: store as `@State`, inject as a stored property, or surface via an `@Observable` model. Caches that mirror the underlying source (e.g. `PipelineQueue.knownSpeakerNames` mirroring the speakers DB) must wire invalidation from every mutation site in the same PR — see issue #155 → PR #158 → PR #159 for the cautionary tale.
