HTML Native Panel — Architecture Audit and Implementation Plan

Status: Architecture audit reconstructed; verification against the current repository required before implementation.
Target: Termux Launcher fork
Feature: HTML panel integrated into the native page/wall system
Implementation status: Not started

---

1. Objective

Introduce HTML as a native panel in the existing page and wall architecture.

The initial implementation should provide:

- HTML rendering through Android WebView.
- CSS and JavaScript support.
- Scrolling and user interaction.
- Opening, closing and switching between pages.
- Preserving the WebView while temporarily switching to another page.
- Correct resource cleanup when a page is permanently removed.
- No regressions in existing terminal panels.

The implementation must follow the existing architecture rather than introduce a parallel navigation system.

2. Scope and non-goals

Included in the MVP

- One independent HTML page.
- Native page lifecycle integration.
- WebView hosting and rendering.
- Basic navigation and Back handling.
- Safe handling of temporary hiding and permanent removal.
- Tests for page lifecycle and integration.

Explicitly excluded

- App Mirror.
- MediaProjection.
- Interactive screen mirroring.
- Rewriting TerminalPaneController.
- Replacing the existing wall architecture.
- Turning HTML into an Android widget.
- A general-purpose browser with unrestricted access to arbitrary websites.

These features can be considered separately after the MVP is stable.

3. Existing architecture

The previous repository audit identified the following components.

Page layer

Relevant files:

- "app/src/main/java/com/termux/app/page/Page.java"
- "app/src/main/java/com/termux/app/page/PageHost.java"
- "app/src/main/java/com/termux/app/page/Pages.java"
- "app/src/main/java/com/termux/app/page/PageKind.java"

The Page contract includes:

- "kind()"
- "side()"
- "view()"
- "rest()"
- "onFront()"
- "offFront()"
- "onBack()"
- Optional glass declaration.

The page system is oriented around the main thread.

Wall layer

Relevant files:

- "wall/PaneWallController.java"
- "wall/PaneWallLayout.java"
- "wall/WallPages.java"
- "wall/WallPagesStore.java"

Previously observed behavior:

- "PaneWallLayout" stores page views by "PageKind".
- "setPageView()" and "pageView()" provide view registration and lookup.
- "PaneWallController" has specialized registration paths for Home, Display, Terminal and Assistant.
- "syncRegistered()" creates pages through factories and synchronizes them with the page collection.
- There is no confirmed generic HTML registration path.

The exact current implementation must be checked again before code changes.

4. PageKind integration

The previously inspected enum was:

public enum PageKind {
    HOME, TERMINAL, DISPLAY, ASSISTANT;

    public boolean hasOwnRoom() {
        return this != DISPLAY;
    }

    public PageKind roomKind() {
        return hasOwnRoom() ? this : TERMINAL;
    }

    public Side side() {
        return this == HOME ? Side.HOME : Side.TERMINAL;
    }
}

Adding "HTML" is not just an enum change.

Every consumer of "PageKind" must be checked for assumptions about the existing values.

Potentially affected areas:

- "LayoutKind"
- "PageRef"
- "PaneWallPolicy"
- "WallRow"
- "PlaceLayoutStore"
- "PageOverviewPages"
- "PageBarModel"
- "PageGlyphSlots"
- "PageOverviewPolicy"
- "PageOverviewAddPolicy"
- "WallKeys"
- "WallPages"
- "TermuxActivity"
- Keyboard, insets, gestures and page overview integration.

Important previously identified risks:

- Unknown page kinds may silently fall back to Terminal.
- Some code branches distinguish only Home, Display and Assistant.
- "WallPages.startIn()" has explicit handling for some kinds and a default path.
- Serialization may depend on enum names.
- Uniqueness rules for HTML pages must be explicit.

Decision: Do not introduce "PageKind.HTML" until all relevant consumers have been audited.

5. Lifecycle — critical requirement

This is the most important architectural constraint.

The previous audit established:

- "Pages.remove(pageId)" removes a page from internal collections.
- It does not directly invoke a destruction callback.
- It does not necessarily call "offFront()" immediately.
- An applied page may still need to receive "offFront()" after removal.
- "Pages.applyFront()" handles the transition between the old and new front pages.
- A test named "PagesTest.aRemovedFrontStillGetsItsOffFront()" confirms this behavior.

Therefore:

Permanent WebView destruction must not be tied directly to "offFront()".

Switching pages is temporary hiding, not permanent removal.

Likewise, detaching a view from its parent does not necessarily mean that its page has been destroyed.

The implementation needs an explicit permanent-release mechanism.

The release process must guarantee:

1. The page is permanently removed.
2. The page is no longer the applied/front page.
3. The WebView is no longer needed by the wall.
4. Any pending transition or attachment has completed safely.
5. The WebView and associated resources are released exactly once.

The final mechanism must be chosen after inspecting the current removal and transition implementation.

6. Proposed component structure

Initial conceptual structure:

PaneWallController
       |
       v
    HtmlPage
       |
       v
  HtmlPanelView
       |
       v
     WebView

Responsibilities:

HtmlPage

- Implements the existing Page contract.
- Represents one HTML page.
- Connects page lifecycle to the HTML view.
- Does not own unrelated wall or terminal behavior.

HtmlPanelView

- Hosts the WebView.
- Handles visibility and attachment.
- Provides a controlled interface for HTML loading and navigation.

WebView

- Renders HTML, CSS and JavaScript.
- Maintains its state while temporarily hidden.
- Is released only after permanent page removal.

This is a proposed design, not a finalized API.

7. WebView requirements

The MVP must use conservative WebView settings.

Required decisions and checks:

- JavaScript should be enabled only where the HTML application requires it.
- File access must be explicitly controlled.
- Mixed-content access must not be enabled without a demonstrated requirement.
- External navigation must have a defined policy.
- Local content loading must use a controlled source.
- Back handling must not unexpectedly navigate away from the entire launcher.
- WebView state should survive ordinary page switching.
- Cleanup must release callbacks, clients and other retained references.

The content source is not finalized.

Possible sources:

- Local HTML files.
- App-provided HTML.
- Explicitly permitted URLs.

The MVP should not silently become an unrestricted browser.

8. Performance requirements

The implementation must follow the repository performance guidelines.

At rest:

- No unnecessary polling.
- No continuous timers without a real requirement.
- No unnecessary redraw loops.
- No repeated expensive work while the page is hidden.

Main thread:

- No file I/O or heavy decoding.
- No expensive work in drawing or touch callbacks.
- UI operations must remain on the appropriate thread.

Resource management:

- Release WebView resources on permanent removal.
- Avoid retaining Activity references.
- Avoid duplicate WebView instances for the same page.
- Measure memory usage and switching behavior.

Physical-device testing requires a separate explicit test step.

9. Implementation phases

Phase A — Complete architecture audit

- Recheck Page and Pages contracts.
- Recheck wall registration and removal.
- Find every PageKind consumer.
- Inspect page overview integration.
- Inspect persistence and serialization.
- Inspect Activity lifecycle and transition behavior.
- Identify all required and conditional files.

Exit condition: A complete file-by-file change map.

Phase B — Finalize architecture

Decide:

- Whether HTML gets a dedicated PageKind.
- Whether one or multiple HTML pages are supported.
- How HTML pages are created.
- How content is selected and loaded.
- How permanent release is coordinated with page removal.

Exit condition: Architecture decisions documented before implementation.

Phase C — Minimal HTML panel

- Add the HTML page implementation.
- Add the WebView host.
- Implement basic loading.
- Implement scrolling and interaction.
- Keep the implementation independent of terminal internals.

Exit condition: HTML renders without disturbing existing pages.

Phase D — Wall integration

- Register the page with the wall.
- Integrate page creation and removal.
- Add page overview representation.
- Implement switching and Back behavior.
- Implement safe permanent cleanup.

Exit condition: HTML behaves like a normal native page.

Phase E — Tests

Test:

- Page creation.
- Page switching.
- Returning to a hidden HTML page.
- Permanent removal.
- Removing the currently visible HTML page.
- Repeated creation and removal.
- Existing terminal page behavior.
- Persistence and restoration, if supported.

Exit condition: No lifecycle regressions or resource leaks identified.

Phase F — Documentation and handoff

- Update architecture documentation.
- Document WebView security decisions.
- Document lifecycle behavior.
- Record known limitations.
- Commit the implementation in logical increments.

10. Acceptance criteria

The MVP is accepted when:

- HTML is available as a native page.
- HTML/CSS/JavaScript work as intended.
- Scrolling and interaction work.
- Switching away does not destroy the page.
- Returning restores the existing WebView state.
- Permanent removal releases resources safely.
- Terminal pages continue to work.
- No unrelated widget or terminal architecture is rewritten.
- Tests cover the important lifecycle transitions.

11. Open decisions

These questions must be resolved before implementation:

1. Should HTML support one page or multiple independent pages?
2. Should the initial content be a local file, app-provided HTML, or an explicitly supplied URL?
3. How should users create a new HTML page from the page overview?
4. Should HTML pages be restored after application restart?
5. What navigation restrictions should apply to external links?

12. Current status

Area| Status
Initial page architecture audit| Previously inspected
Wall architecture| Previously inspected
PageKind impact| Partially inspected
Page lifecycle| Critical behavior identified
WebView design| Proposed
File-by-file change map| Pending
Source implementation| Not started
Tests| Not started
Device validation| Not started

13. Working rules

- Do not modify source code before Phase A and Phase B are complete.
- Do not rewrite TerminalPaneController.
- Do not treat HTML as a widget.
- Do not destroy WebView on ordinary page switching.
- Do not assume that detachment means permanent removal.
- Verify repository behavior before relying on reconstructed audit notes.
- Keep documentation and implementation history understandable to another developer or AI assistant.
- Use small, reviewable commits with explanatory messages.

---

Next action: Re-audit the current repository and produce the verified file-by-file change map before touching source code.
