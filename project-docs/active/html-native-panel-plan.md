HTML Native Panel — Implementation Plan

Status: Planning
Base: "upstream/pre", "e521bee3d" ("v2.0.0-pre.7")
Feature branch: "feat/native-panels"
Scope: Standalone native HTML panel in the existing page wall

1. Goal

Implement a standalone HTML panel inside the existing wall/page system.

The panel must:

- Render HTML using Android WebView.
- Support HTML, CSS and JavaScript.
- Support scrolling and normal page interaction.
- Open, close and switch like other wall pages.
- Preserve WebView state when temporarily leaving the panel.
- Release WebView resources after permanent removal.
- Coexist with terminal pages without changing terminal pane architecture.

The HTML panel is a native page, not a widget.

2. Explicit non-goals

The first milestone does not include:

- App Mirror or MediaProjection.
- Interactive screen mirroring.
- A general-purpose HTML widget framework.
- Integration with "WidgetPaneView".
- Changes to "TerminalPaneController".
- Multiple independent HTML pages.
- A full browser with tabs, bookmarks or download management.
- A remote web application platform.

These may be considered separately after the first milestone works.

3. Existing architecture

3.1 Page contract

Relevant files:

- "page/Page.java"
- "page/PageHost.java"
- "page/Pages.java"
- "page/PageKind.java"

"Page" already defines the main page lifecycle:

- "kind()"
- "side()"
- "view()"
- "rest()"
- "onFront()"
- "offFront()"
- "onBack()"

The interface is main-thread oriented.

"PageHost" is deliberately small and does not expose the Activity directly.

3.2 Wall and view registration

Relevant files:

- "wall/PaneWallController.java"
- "wall/PaneWallLayout.java"
- "wall/WallPages.java"
- "wall/WallPagesStore.java"

"PaneWallLayout" maintains an "EnumMap<PageKind, View>" and exposes page-view registration.

"PaneWallController" currently registers Home, Display, Terminal and Assistant through specialized paths.

A new page type will need an explicit registration path. The existence of the generic View map does not mean the rest of the wall is fully generic.

3.3 Page storage and policies

A new "PageKind" affects more than the enum itself.

Known integration points include:

- "WallPages" creation, uniqueness, removal and start-in behavior.
- "WallPagesStore" serialization and restoration.
- "PaneWallController" page synchronization.
- "PaneWallLayout" view and layout behavior.
- "WallKeys" extra-key registration.
- "PageOverviewPages" card mapping and page operations.
- "PageOverviewPolicy" card behavior and removal.
- "LayoutKind" selection.
- Page navigation, Back handling and other "PageKind" consumers.

Unknown page kinds must not silently fall through to Terminal or Home behavior.

4. Proposed component structure

Initial conceptual structure:

PaneWallController
    |
    +-- HTML page registration
            |
            +-- HtmlPage
                    |
                    +-- HtmlPanelView
                            |
                            +-- Android WebView

Responsibilities:

HtmlPage

- Implements the existing "Page" contract.
- Owns the HTML page's lifecycle.
- Exposes its View.
- Handles front/back transitions.
- Coordinates final resource release.

HtmlPanelView

- Owns the WebView.
- Provides the panel's visual container.
- Handles WebView attachment and detachment.
- Does not own wall navigation or page storage.

WebView

- Renders the supplied HTML.
- Handles CSS, JavaScript and user interaction.
- Must not be recreated on every page switch.

PaneWallController

- Creates and registers the HTML page.
- Synchronizes its presence with the wall's page list.
- Removes its view when the page is permanently removed.

This is a proposed separation of responsibilities, not a final class/API commitment.

5. PageKind decision

The likely direction is a dedicated "PageKind.HTML".

Before adding it, audit every relevant consumer of "PageKind".

In particular:

- "hasOwnRoom()" and "roomKind()".
- "side()".
- "LayoutKind.of()".
- "PageOverviewPolicy".
- "PageOverviewPages.cardKind()".
- "WallKeys".
- "WallPages.startIn()".
- Page navigation and Back behavior.
- Serialization and restoration.

Define explicitly:

- Whether HTML has its own room.
- Which side it belongs to.
- Which layout it uses.
- Whether it appears in the extra-keys row.
- How it appears in the page overview.
- Whether it can be moved or removed.
- Whether only one HTML page is allowed.

Do not add the enum value until these behaviors are understood.

6. Critical lifecycle requirement

The existing lifecycle has an important distinction between registry removal and page deactivation.

"Pages.remove(pageId)" removes the page from its registry, but does not directly call "offFront()".

"Pages" may still retain the removed page as "mApplied". The existing test "PagesTest.aRemovedFrontStillGetsItsOffFront()" verifies that it receives "offFront()" when the next front is applied.

"WallPagesStore.Listener.onRemoved()" is the final-removal notification, but it currently has no generic page destruction mechanism.

Therefore, the HTML implementation must distinguish:

1. Active
   
   - The HTML page is in front.
   - WebView is available for interaction.

2. Inactive
   
   - The user has navigated away.
   - WebView remains alive.
   - Temporary work can be paused or reduced.

3. Pending final release
   
   - The page has been permanently removed.
   - It must not be recreated or reused.
   - Release waits until the page is no longer applied or attached.

4. Released
   
   - WebView resources are destroyed.
   - No callbacks should attempt to use the released WebView.

The final-release path must account for wall transitions and delayed "offFront()" calls.

Do not destroy WebView merely because "offFront()" was called.

Do not assume that removing a View from the page registry automatically detaches it from its parent.

7. WebView requirements

The initial implementation should explicitly define:

- JavaScript enabled only where required by the intended HTML content.
- File access policy.
- Content access policy.
- Mixed-content policy.
- Navigation handling.
- External links behavior.
- Error handling.
- Back navigation behavior.
- WebView state retention.
- Cleanup and destruction.

Security-sensitive WebView settings must be conservative by default.

Do not enable universal file access or unrestricted navigation as a shortcut.

The initial content source must be specified before implementing the loading interface.

8. Performance requirements

Follow "docs/agents/performance.md".

Requirements:

- Do not continuously reload HTML when switching pages.
- Do not create a new WebView on every "onFront()".
- Do not perform file or network I/O on the main thread.
- Avoid unnecessary WebView instances.
- Stop unnecessary background work while inactive.
- Release resources on permanent removal.
- Keep terminal page behavior unchanged.

Measure memory and transition behavior after the first functional implementation.

9. Implementation phases

Phase A — Complete architecture audit

No source edits.

1. Enumerate all "PageKind" consumers.
2. Inspect page overview and navigation policies.
3. Inspect "WallPagesStore" serialization and removal ordering.
4. Inspect page-view attachment and detachment.
5. Inspect how Assistant page creation and final cleanup are wired.
6. Identify the safest ownership point for WebView destruction.
7. Confirm how the HTML page's content will be supplied.

Exit condition: every required integration point and lifecycle transition is understood.

Phase B — Architecture decision

Document:

- Whether to add "PageKind.HTML".
- HTML page uniqueness rules.
- Page factory and registration path.
- Page overview behavior.
- WebView ownership.
- Final-release mechanism.
- Content source and loading contract.

Exit condition: no unresolved ownership ambiguity.

Phase C — Minimal native panel

Implement:

- HTML page class.
- HTML panel View.
- WebView creation.
- Basic HTML loading.
- Page registration.
- Open and close behavior.
- Switching without unnecessary reloads.
- Safe final release.

Do not add unrelated browser functionality.

Phase D — Wall integration

Implement:

- Page storage and restoration.
- Page overview card.
- Add/remove behavior.
- Back handling.
- Explicit layout and navigation policy.
- Correct wall synchronization.

Verify terminal pages remain unaffected.

Phase E — Tests and validation

Add focused tests for:

- Page creation and registration.
- Page switching.
- Active page removal.
- Inactive page removal.
- Removal during a wall transition.
- Final WebView release.
- Restored page state.
- Page overview behavior.
- Existing terminal behavior.

Run relevant unit tests and build checks.

Phase F — Documentation and handoff

Update:

- Active HTML panel specification.
- Architecture decisions.
- Lifecycle explanation.
- Test coverage.
- Remaining limitations.

Move the specification to reference documentation only when the implementation is complete and stable.

10. Acceptance criteria

The first milestone is complete when:

- [ ] HTML panel exists as a standalone wall page.
- [ ] HTML/CSS/JavaScript render correctly.
- [ ] Scrolling and interaction work.
- [ ] Panel can be opened.
- [ ] Panel can be closed.
- [ ] Switching away and back does not unnecessarily recreate WebView.
- [ ] Permanent removal releases WebView safely.
- [ ] No lifecycle crash occurs during page transitions.
- [ ] Terminal panes remain unchanged.
- [ ] Relevant tests pass.
- [ ] Documentation describes the actual implementation.

11. Current status

Completed:

- Initial page/wall architecture audit.
- "Page" lifecycle inspection.
- "Pages" removal behavior inspection.
- "WallPagesStore" final-removal notification inspection.
- Initial review of page overview policies.
- Review of Assistant and Display page patterns.
- Review of performance requirements.

Not started:

- Source code changes.
- New "PageKind".
- WebView implementation.
- UI integration.
- Tests.
- Commits.

12. Open decisions

1. What is the initial HTML content source: local HTML file, URL, or HTML supplied by another part of the app?
2. Should the first version expose one HTML page or eventually support multiple independent HTML pages?
3. What exact page overview action creates the HTML panel?

These decisions must be settled before implementing the loading UI and final page policies.
