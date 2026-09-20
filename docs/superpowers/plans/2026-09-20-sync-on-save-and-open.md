# Sync on Save + Sync on Open (Issue #65) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add two opt-in settings that trigger a full sync automatically — one when vault files are saved (debounced), one when Obsidian launches — per GitHub issue [joshuakto/fit#65](https://github.com/joshuakto/fit/issues/65).

**Architecture:** All logic lives in `FitPlugin` (the Obsidian integration layer), matching the existing auto-sync interval timer. A `vault.on('modify')` listener arms a 3-second debounce that calls the existing `executeSyncWithUICoordination('auto')`; `onload()` fires the same entry point once at launch. No new `FitSync` operations, no new `LocalStores` fields — both triggers reuse the full sync (pull + push) path.

**Tech Stack:** TypeScript, Obsidian plugin API (`vault.on`, `registerEvent`), Vitest (happy-dom), existing `fitPlugin.test.ts` / `fitSetting.test.ts` test harnesses.

**Spec:** Design approved in-session on 2026-09-20 (chat): full sync for both triggers (not pull-only/push-only), both settings independent of the `autoSync` dropdown, both default off, 3s hardcoded debounce, save events dropped while `fitSync.isActive` (covers FIT's own pull writes firing `modify`).

## Global Constraints

- **Mobile-safe only:** no Node.js built-ins; `vault.on()` and `Plugin.registerEvent()` are public Obsidian APIs safe on all platforms.
- **Both new settings default to `false`** (opt-in; design principle "user agency").
- **Both settings are independent of the `autoSync` dropdown** — `autoSync` keeps controlling only the interval timer.
- **Full sync only:** both triggers call `executeSyncWithUICoordination('auto')`; no new `FitSync` method.
- **Never lose data:** a save that lands while a sync is in flight is dropped by the `isActive` guard, not queued — the change stays pending locally and ships on the next trigger.
- **Verification commands:** `npm test` (full), `npm test -- --testNamePattern="..."` (targeted), `npm run typecheck && npm run lint` (required before declaring a task done).
- **Test conventions** (docs/CONTRIBUTING.md): one test = one thing; mock only true external boundaries (Obsidian API via `src/__mocks__/obsidian.ts`), not internal modules.
- **Code style:** tab indentation; arrow-function class fields for handlers that use `this` (see existing `triggerManualSync`).
- **README reflects the stable release** (currently 1.5.0): the new feature gets a "Coming soon" note, not a feature bullet.

## File Structure

| File | Change | Responsibility delta |
|---|---|---|
| `src/fitSettings.ts` | Modify | Add `syncOnSave`, `syncOnOpen` fields + defaults |
| `src/fitPlugin.ts` | Modify | Event registration, save-debounce handler, open trigger, `onload`/`onunload` wiring, `loadSettings` boolean coercion |
| `src/fitSettingTab.ts` | Modify | Two toggles in `localConfigBlock()` ("Local configurations") |
| `src/__mocks__/obsidian.ts` | Modify | Add `registerEvent` to the Plugin mock |
| `src/fitPlugin.test.ts` | Modify | Trigger-handler tests + `loadSettings` coercion tests |
| `src/fitSetting.test.ts` | Modify | Toggle rendering/persistence test |
| `docs/architecture.md` | Modify | FitPlugin bullet mentions event-driven triggers |
| `docs/CONTRIBUTING.md` | Modify | Roadmark line: on-save/on-open shipped |
| `README.md` | Modify | "Coming soon" note |

---

### Task 1: Settings fields + loadSettings coercion

**Files:**
- Modify: `src/fitSettings.ts:23` (interface), `src/fitSettings.ts:41` (defaults)
- Modify: `src/fitPlugin.ts:536` (boolean coercion list in `loadSettings`)
- Test: `src/fitPlugin.test.ts` (existing `describe('loadSettings from disk')` block, ~line 198)

**Interfaces:**
- Produces: `FitSettings.syncOnSave: boolean`, `FitSettings.syncOnOpen: boolean` (default `false`) — consumed by Tasks 2-4.

- [ ] **Step 1: Write the failing test**

In `src/fitPlugin.test.ts`, inside the existing `describe('loadSettings from disk')` block, add:

```ts
	it('defaults syncOnSave and syncOnOpen to false', async () => {
		const plugin = makePlugin();
		mockLoad(plugin, {});
		await plugin.loadSettings();
		expect(plugin.settings.syncOnSave).toBe(false);
		expect(plugin.settings.syncOnOpen).toBe(false);
	});

	it('keeps stored syncOnSave/syncOnOpen values', async () => {
		const plugin = makePlugin();
		mockLoad(plugin, { syncOnSave: true, syncOnOpen: true });
		await plugin.loadSettings();
		expect(plugin.settings.syncOnSave).toBe(true);
		expect(plugin.settings.syncOnOpen).toBe(true);
	});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- --testNamePattern="loadSettings from disk"`
Expected: the two new tests FAIL (`undefined` is not `false`/`true`); pre-existing tests still pass.

- [ ] **Step 3: Implement the settings fields**

In `src/fitSettings.ts`, add to the `FitSettings` interface after `syncHiddenFiles: boolean`:

```ts
	syncOnSave: boolean
	syncOnOpen: boolean
```

In `DEFAULT_SETTINGS`, add after `syncHiddenFiles: true,`:

```ts
	syncOnSave: false,
	syncOnOpen: false,
```

In `src/fitPlugin.ts` `loadSettings()` (line ~536), add both keys to the boolean-coercion branch:

```ts
					else if (key === "notifyChanges" || key === "notifyConflicts" || key === "enableDebugLogging" || key === "syncHiddenFiles" || key === "syncOnSave" || key === "syncOnOpen") {
						obj[key] = Boolean(settings[key]);
					}
```

- [ ] **Step 4: Run tests and typecheck**

Run: `npm test -- --testNamePattern="loadSettings from disk" && npm run typecheck`
Expected: all `loadSettings from disk` tests PASS, typecheck clean.

- [ ] **Step 5: Commit**

```bash
git add src/fitSettings.ts src/fitPlugin.ts src/fitPlugin.test.ts
git commit -m "feat: add syncOnSave/syncOnOpen settings fields (#65)"
```

---

### Task 2: Sync-on-save handler (debounced vault `modify` listener)

**Files:**
- Modify: `src/fitPlugin.ts` (import line 1; new field near line 64; new handler after `handleAutoSyncTimer`, ~line 451)
- Test: `src/fitPlugin.test.ts` (new top-level describe)

**Interfaces:**
- Consumes: `FitSettings.syncOnSave` (Task 1), `FitSync.isActive`, `executeSyncWithUICoordination('auto')`.
- Produces: `onVaultFileSaved(file: TFile): void` (debounce constant `SAVE_SYNC_DEBOUNCE_MS = 3000`, field `saveSyncDebounceTimer: number | null`) — consumed by Task 3 (registration + onunload cleanup).

- [ ] **Step 1: Write the failing tests**

In `src/fitPlugin.test.ts`, add a new top-level describe block:

```ts
describe('FitPlugin sync-on-save trigger', () => {
	// Vault 'modify' events are debounced: back-to-back saves coalesce into one
	// full auto sync. FIT's own pull writes also fire 'modify' (LocalVault uses
	// vault.modify/create), so the isActive guard must prevent re-triggering.
	const SAVE_DEBOUNCE_MS = 3000; // must match SAVE_SYNC_DEBOUNCE_MS in fitPlugin.ts

	function makeSaveTriggerPlugin(settingsOverride: Partial<typeof DEFAULT_SETTINGS> = {}) {
		const plugin = makePlugin();
		plugin.settings = {
			...DEFAULT_SETTINGS,
			pat: 'token', owner: 'alice', repo: 'notes', branch: 'main',
			notifyChanges: false,
			notifyConflicts: false,
			...settingsOverride,
		};
		plugin.fit = { loadLocalStore: vi.fn(), loadSettings: vi.fn() } as any;
		plugin.fitSyncRibbonIconEl = { addClass: vi.fn(), removeClass: vi.fn() } as any;
		(plugin.fitSync as unknown as StubFitSync).sync.mockResolvedValue({
			success: true, changeGroups: [], clash: [],
		});
		return plugin;
	}

	const fakeFile = { path: 'notes/a.md' };

	beforeEach(() => {
		vi.useFakeTimers();
	});

	afterEach(() => {
		vi.useRealTimers();
	});

	it('triggers a full auto sync after the debounce window', async () => {
		const plugin = makeSaveTriggerPlugin({ syncOnSave: true });
		const stub = plugin.fitSync as unknown as StubFitSync;

		(plugin as any).onVaultFileSaved(fakeFile);
		await vi.advanceTimersByTimeAsync(SAVE_DEBOUNCE_MS - 1);
		expect(stub.sync).not.toHaveBeenCalled();

		await vi.advanceTimersByTimeAsync(1);
		expect(stub.sync).toHaveBeenCalledTimes(1);
		expect(stub.sync).toHaveBeenCalledWith(expect.anything(), { isAutoSync: true });
	});

	it('is a no-op when syncOnSave is disabled', async () => {
		const plugin = makeSaveTriggerPlugin({ syncOnSave: false });
		const stub = plugin.fitSync as unknown as StubFitSync;

		(plugin as any).onVaultFileSaved(fakeFile);
		await vi.advanceTimersByTimeAsync(SAVE_DEBOUNCE_MS);

		expect(stub.sync).not.toHaveBeenCalled();
	});

	it('is a no-op while a sync is already in progress', async () => {
		const plugin = makeSaveTriggerPlugin({ syncOnSave: true });
		const stub = plugin.fitSync as unknown as StubFitSync;
		stub.isActive = true;

		(plugin as any).onVaultFileSaved(fakeFile);
		await vi.advanceTimersByTimeAsync(SAVE_DEBOUNCE_MS);

		expect(stub.sync).not.toHaveBeenCalled();
	});

	it('coalesces rapid successive saves into a single sync', async () => {
		const plugin = makeSaveTriggerPlugin({ syncOnSave: true });
		const stub = plugin.fitSync as unknown as StubFitSync;

		(plugin as any).onVaultFileSaved(fakeFile);
		await vi.advanceTimersByTimeAsync(1000);
		(plugin as any).onVaultFileSaved(fakeFile);
		await vi.advanceTimersByTimeAsync(SAVE_DEBOUNCE_MS);

		expect(stub.sync).toHaveBeenCalledTimes(1);
	});

	it('onunload clears a pending debounce timer', async () => {
		const plugin = makeSaveTriggerPlugin({ syncOnSave: true });
		const stub = plugin.fitSync as unknown as StubFitSync;

		(plugin as any).onVaultFileSaved(fakeFile);
		plugin.onunload();
		await vi.advanceTimersByTimeAsync(SAVE_DEBOUNCE_MS);

		expect(stub.sync).not.toHaveBeenCalled();
	});
});
```

Note: `beforeEach`/`afterEach` are already imported at the top of the file. The `'onunload clears a pending debounce timer'` test intentionally belongs here (it exercises the save-trigger timer, not plugin unload generally).

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- --testNamePattern="sync-on-save trigger"`
Expected: all 5 FAIL with `TypeError: (plugin ...).onVaultFileSaved is not a function` (the onunload test fails with `onVaultFileSaved is not a function` too).

- [ ] **Step 3: Implement the handler**

In `src/fitPlugin.ts`:

1. Extend the obsidian import (line 1) to include `TFile`:

```ts
import { Notice, Plugin, SettingTab, TFile } from 'obsidian';
```

2. Add a module-level constant just above the class declaration (before `export default class FitPlugin`):

```ts
const SAVE_SYNC_DEBOUNCE_MS = 3000;
```

3. Add a field after `private currentSyncNotice: FitNotice | null = null;`:

```ts
	private saveSyncDebounceTimer: number | null = null; // Pending debounced sync after a file save
```

4. Add the handler after `handleAutoSyncTimer()` (before `startOrUpdateAutoSyncInterval`):

```ts
	/**
	 * Entry point: a vault file was written to disk (Ctrl+S, vim :w, editor
	 * autosave on blur, or a mobile save). Debounced so a burst of saves
	 * coalesces into one full auto sync. The isActive guard is load-bearing:
	 * FIT's own pull writes fire 'modify' too (LocalVault.applyChanges uses
	 * vault.modify/create), so without it every sync would re-arm a redundant
	 * no-op sync a few seconds later.
	 */
	onVaultFileSaved = (_file: TFile): void => {
		if (!this.settings?.syncOnSave || this.fitSync?.isActive) return;

		if (this.saveSyncDebounceTimer !== null) {
			window.clearTimeout(this.saveSyncDebounceTimer);
		}
		this.saveSyncDebounceTimer = window.setTimeout(() => {
			this.saveSyncDebounceTimer = null;
			void this.executeSyncWithUICoordination('auto');
		}, SAVE_SYNC_DEBOUNCE_MS);
	};
```

5. In `onunload()`, clear the pending timer (before or after the interval cleanup):

```ts
	onunload() {
		if (this.saveSyncDebounceTimer !== null) {
			window.clearTimeout(this.saveSyncDebounceTimer);
			this.saveSyncDebounceTimer = null;
		}
		if (this.autoSyncIntervalId !== null) {
			window.clearInterval(this.autoSyncIntervalId);
			this.autoSyncIntervalId = null;
		}
	}
```

- [ ] **Step 4: Run tests, typecheck, and lint**

Run: `npm test -- --testNamePattern="sync-on-save trigger" && npm run typecheck && npm run lint`
Expected: all 5 new tests PASS, typecheck clean, lint clean.

- [ ] **Step 5: Commit**

```bash
git add src/fitPlugin.ts src/fitPlugin.test.ts
git commit -m "feat: debounced sync-on-save trigger via vault modify events (#65)"
```

---

### Task 3: Sync-on-open trigger + event registration wiring

**Files:**
- Modify: `src/fitPlugin.ts` (`onload` ~line 508, new methods)
- Modify: `src/__mocks__/obsidian.ts:51-61` (Plugin mock)
- Test: `src/fitPlugin.test.ts` (new top-level describe)

**Interfaces:**
- Consumes: `FitSettings.syncOnOpen` (Task 1), `onVaultFileSaved` (Task 2), `checkSettingsConfigured()`, `executeSyncWithUICoordination('auto')`.
- Produces: `handleSyncOnOpen(): Promise<void>`, `registerVaultEvents(): void` — wired into `onload()` in this task.

- [ ] **Step 1: Write the failing tests**

In `src/fitPlugin.test.ts`, add a new top-level describe block:

```ts
describe('FitPlugin sync-on-open trigger', () => {
	// One-shot full sync at app launch when the user opted in. Mirrors
	// handleAutoSyncTimer's guard structure (independent of the autoSync
	// dropdown; unconfigured vaults get the standard settings prompt).
	function makeOpenTriggerPlugin(settingsOverride: Partial<typeof DEFAULT_SETTINGS> = {}) {
		const plugin = makePlugin();
		plugin.settings = {
			...DEFAULT_SETTINGS,
			notifyChanges: false,
			notifyConflicts: false,
			...settingsOverride,
		};
		plugin.fit = { loadLocalStore: vi.fn(), loadSettings: vi.fn() } as any;
		plugin.fitSyncRibbonIconEl = { addClass: vi.fn(), removeClass: vi.fn() } as any;
		(plugin.fitSync as unknown as StubFitSync).sync.mockResolvedValue({
			success: true, changeGroups: [], clash: [],
		});
		return plugin;
	}

	const configured = { pat: 'token', owner: 'alice', repo: 'notes', branch: 'main' };

	it('runs a full auto sync at open when enabled and configured', async () => {
		const plugin = makeOpenTriggerPlugin({ ...configured, syncOnOpen: true });
		const stub = plugin.fitSync as unknown as StubFitSync;

		await (plugin as any).handleSyncOnOpen();

		expect(stub.sync).toHaveBeenCalledTimes(1);
		expect(stub.sync).toHaveBeenCalledWith(expect.anything(), { isAutoSync: true });
	});

	it('does nothing when syncOnOpen is disabled', async () => {
		const plugin = makeOpenTriggerPlugin({ ...configured, syncOnOpen: false });
		const stub = plugin.fitSync as unknown as StubFitSync;

		await (plugin as any).handleSyncOnOpen();

		expect(stub.sync).not.toHaveBeenCalled();
	});

	it('does nothing when settings are not configured', async () => {
		const plugin = makeOpenTriggerPlugin({ syncOnOpen: true }); // no pat/owner/repo/branch
		plugin.app.setting = { open: vi.fn(), openTabById: vi.fn() } as any;
		const stub = plugin.fitSync as unknown as StubFitSync;

		await (plugin as any).handleSyncOnOpen();

		expect(stub.sync).not.toHaveBeenCalled();
	});
});
```

And in the `describe('FitPlugin sync-on-save trigger')` block from Task 2, add one more test (it exercises the on-save listener registration):

```ts
	it('registerVaultEvents subscribes the save handler to vault modify events', () => {
		const plugin = makeSaveTriggerPlugin({ syncOnSave: true });
		plugin.app.vault = { on: vi.fn().mockReturnValue({}) } as any;

		(plugin as any).registerVaultEvents();

		expect(plugin.app.vault.on).toHaveBeenCalledWith('modify', expect.any(Function));
		expect(plugin.registerEvent).toHaveBeenCalledTimes(1);
	});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test -- --testNamePattern="sync-on-open trigger"`
Expected: 3 FAIL (`handleSyncOnOpen is not a function`).
Run: `npm test -- --testNamePattern="sync-on-save trigger"`
Expected: the new registration test FAILS (`registerVaultEvents is not a function`); Task 2's tests still pass.

- [ ] **Step 3: Implement the handler, registration, and onload wiring**

1. In `src/__mocks__/obsidian.ts`, add to the `Plugin` class (after `registerDomEvent`):

```ts
	registerEvent = vi.fn();
```

2. In `src/fitPlugin.ts`, add two methods after `handleSyncOnOpen`'s natural neighbor — place both after `handleAutoSyncTimer()` (order within the file: `handleAutoSyncTimer`, `handleSyncOnOpen`, `registerVaultEvents`, `startOrUpdateAutoSyncInterval`):

```ts
	/**
	 * Entry point: app launch — one-shot full sync when the user opted in
	 * ("Sync on open", #65). Independent of the autoSync interval setting.
	 * Mirrors handleAutoSyncTimer's guard structure, including the
	 * not-configured prompt.
	 */
	async handleSyncOnOpen(): Promise<void> {
		if (!this.settings?.syncOnOpen) return;
		if (this.checkSettingsConfigured()) {
			await this.executeSyncWithUICoordination('auto');
		}
	}

	/**
	 * Register vault event listeners driving the auto-sync triggers.
	 * registerEvent() auto-unregisters them on plugin unload.
	 */
	registerVaultEvents(): void {
		this.registerEvent(this.app.vault.on('modify', this.onVaultFileSaved));
	}
```

3. In `onload()`, after the `await this.startOrUpdateAutoSyncInterval();` line (~line 508), add:

```ts
			this.registerVaultEvents();

			// One-shot sync at launch when opted in; not awaited so a slow
			// network never delays vault load.
			void this.handleSyncOnOpen();
```

- [ ] **Step 4: Run tests, typecheck, and lint**

Run: `npm test -- --testNamePattern="trigger" && npm run typecheck && npm run lint`
Expected: all sync-on-save + sync-on-open tests PASS, typecheck clean, lint clean.

- [ ] **Step 5: Commit**

```bash
git add src/fitPlugin.ts src/__mocks__/obsidian.ts src/fitPlugin.test.ts
git commit -m "feat: one-shot sync-on-open trigger and vault event registration (#65)"
```

---

### Task 4: Settings UI toggles

**Files:**
- Modify: `src/fitSettingTab.ts:692-696` (in `localConfigBlock()`, between the auto-check-interval slider block and the "Sync hidden files" toggle)
- Test: `src/fitSetting.test.ts` (new top-level describe)

**Interfaces:**
- Consumes: `FitSettings.syncOnSave` / `syncOnOpen` (Task 1), `plugin.saveSettings()`.
- Produces: user-visible "Sync on save" and "Sync on open" toggles in the "Local configurations" section.

- [ ] **Step 1: Write the failing test**

In `src/fitSetting.test.ts`, add a new top-level describe (reuses the file's existing helpers/import style — `FitSettingTab`, `DEFAULT_SETTINGS`, `FitLogger` are already imported):

```ts
describe('FitSettingTab - auto-sync triggers (#65)', () => {
	it('renders sync-on-save/sync-on-open toggles that persist on change', async () => {
		const mockLogger = new FitLogger({ adapter: null });
		const fakePlugin: any = {
			settings: { ...DEFAULT_SETTINGS, syncOnSave: false, syncOnOpen: false },
			saveSettings: vi.fn().mockResolvedValue(undefined),
			logger: mockLogger,
		};

		const settingTab = new FitSettingTab({} as any, fakePlugin);
		settingTab.localConfigBlock();

		const findToggleByLabel = (labelText: string): HTMLInputElement | null => {
			const settings = Array.from(settingTab.containerEl.querySelectorAll('.setting-item'));
			for (const setting of settings) {
				const nameEl = setting.querySelector('.setting-item-name');
				if (nameEl?.textContent === labelText) {
					return setting.querySelector('input[type="checkbox"]') as HTMLInputElement | null;
				}
			}
			return null;
		};

		const saveToggle = findToggleByLabel('Sync on save')!;
		const openToggle = findToggleByLabel('Sync on open')!;
		expect(saveToggle.checked).toBe(false);
		expect(openToggle.checked).toBe(false);

		saveToggle.checked = true;
		saveToggle.dispatchEvent(new Event('change'));
		openToggle.checked = true;
		openToggle.dispatchEvent(new Event('change'));

		await vi.waitFor(() => expect(fakePlugin.saveSettings).toHaveBeenCalledTimes(2));
		expect(fakePlugin.settings.syncOnSave).toBe(true);
		expect(fakePlugin.settings.syncOnOpen).toBe(true);
	});
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- --testNamePattern="auto-sync triggers"`
Expected: FAIL — `findToggleByLabel('Sync on save')` returns null (`Cannot read properties of null (reading 'checked')`).

- [ ] **Step 3: Implement the toggles**

In `src/fitSettingTab.ts` `localConfigBlock()`, insert after the `if (this.plugin.settings.autoSync === "off") { checkIntervalSlider.settingEl.addClass("clear"); }` block and before the `// Hidden files setting` comment:

```ts
		new Setting(containerEl)
			.setName("Sync on save")
			.setDesc("Wait a few seconds after a vault file is saved, then run a full sync. Saves made in quick succession trigger a single sync.")
			.addToggle(toggle => toggle
				.setValue(this.plugin.settings.syncOnSave)
				.onChange(async (value) => {
					this.plugin.settings.syncOnSave = value;
					await this.plugin.saveSettings();
				}));

		new Setting(containerEl)
			.setName("Sync on open")
			.setDesc("Run a full sync when Obsidian launches, so remote changes from your other devices are pulled in immediately.")
			.addToggle(toggle => toggle
				.setValue(this.plugin.settings.syncOnOpen)
				.onChange(async (value) => {
					this.plugin.settings.syncOnOpen = value;
					await this.plugin.saveSettings();
				}));
```

- [ ] **Step 4: Run tests, typecheck, and lint**

Run: `npm test -- --testNamePattern="auto-sync triggers" && npm run typecheck && npm run lint`
Expected: new test PASSES, typecheck clean, lint clean.

- [ ] **Step 5: Commit**

```bash
git add src/fitSettingTab.ts src/fitSetting.test.ts
git commit -m "feat: sync-on-save and sync-on-open settings toggles (#65)"
```

---

### Task 5: Docs + full verification gate

**Files:**
- Modify: `docs/architecture.md:28` (FitPlugin bullet)
- Modify: `docs/CONTRIBUTING.md` (roadmap "Auto-sync triggers" line, ~line 65)
- Modify: `README.md` ("Coming soon" section, ~line 30)

**Interfaces:**
- Consumes: nothing (documentation only).

- [ ] **Step 1: Update docs/architecture.md**

In the "### FitPlugin" bullet list, change:

```md
- Manages plugin loading, settings persistence, auto-sync scheduling
```

to:

```md
- Manages plugin loading, settings persistence, auto-sync scheduling (interval timer plus opt-in event triggers: sync on file save, sync on app open)
```

- [ ] **Step 2: Update docs/CONTRIBUTING.md roadmap**

Change the roadmap line:

```md
- **Auto-sync triggers** - On save, on open, configurable intervals
```

to:

```md
- **Auto-sync triggers** - On save / on open (shipped, #65); remaining: per-trigger tuning
```

- [ ] **Step 3: Update README.md "Coming soon"**

In the "## Coming soon" section, add after the "Sync status overview" paragraph:

```md
**Sync on save / sync on open** — opt-in toggles (off by default): a full sync a few seconds after you save a file, and a full sync when Obsidian launches, so changes from your other devices are pulled in immediately. Independent of the periodic auto-sync interval.
```

(Per AGENTS.md: the README reflects the stable release, so this stays under "Coming soon" until 1.6 ships.)

- [ ] **Step 4: Run the full verification gate**

Run: `npm test && npm run typecheck && npm run lint`
Expected: full suite passes, typecheck clean, lint clean.

- [ ] **Step 5: Commit**

```bash
git add docs/architecture.md docs/CONTRIBUTING.md README.md
git commit -m "docs: document sync-on-save and sync-on-open triggers (#65)"
```

---

## Self-Review Notes

- **Spec coverage:** on-save trigger (Task 2), on-open trigger (Task 3), independent of `autoSync` (handlers never read `settings.autoSync`; guards are `syncOnSave`/`syncOnOpen` + config + `isActive`), full sync via existing path (both call `executeSyncWithUICoordination('auto')`), defaults off (Task 1), debounce + coalescing (Task 2 tests 1 and 4), re-trigger suppression via `isActive` (Task 2 test 3), settings UI (Task 4), docs (Task 5).
- **Placeholder scan:** none — every step carries concrete code.
- **Type consistency:** `syncOnSave`/`syncOnOpen` spelled identically across Tasks 1-4; `onVaultFileSaved`, `handleSyncOnOpen`, `registerVaultEvents`, `saveSyncDebounceTimer`, `SAVE_SYNC_DEBOUNCE_MS` defined once (Task 2/3) and reused; the test constant `SAVE_DEBOUNCE_MS` is documented as needing to match the implementation constant.
- **Known accepted behavior:** a save that lands mid-sync is dropped (change stays pending for the next trigger); save/open-triggered syncs show the same transient "Auto syncing" notice behavior as interval auto-sync (muted only when `autoSync === "muted"`), silent on success, sticky error notice on failure.
