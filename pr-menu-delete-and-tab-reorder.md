Title: Make menu delete configurable and add tab reordering commands

Summary
- Expose a generic menu delete action via control_menu so users can bind any key to delete the selected entry in list/table selectors (e.g., goto_tab).
- Add commands to reorder tabs (move current tab left/right) so users can control tab order via keybindings.

Context
- Today, deleting an entry in selectors (like goto_tab) is hard-wired to the Delete key processed in BaseSelectorWidget’s event filter, not configurable via the keybinding system.
- Tab order is stored in DocumentManager as a vector, but there are no commands to reorder tabs; only cycling with goto_next_tab/goto_prev_tab exists.

Scope
1) Menu delete via control_menu
   - Add a new action string handled by MainWidget::handle_action_in_menu that triggers the existing delete pathway:
     - If a BaseSelectorWidget is active and action ∈ {"delete", "remove", "del"}, call selector_widget->handle_delete().
   - This leverages existing on_delete callbacks (e.g., goto_tab’s lambda) with no menu-specific wiring.
   - Documentation: add an example binding in keys.config/comments:
       [m] control_menu(delete) <C-d>

2) Tab reordering commands
   - Add DocumentManager::move_tab(int from, int to) to reorder the internal tabs vector.
   - Add MainWidget helpers to compute current index and call move_tab safely (bounds-checked, no-ops on invalid indices):
       - move_current_tab_left()
       - move_current_tab_right()
   - Add commands (input.cpp):
       - move_current_tab_left (human: Move current tab left)
       - move_current_tab_right (human: Move current tab right)
   - Optional default keybind suggestions (not enforced):
       move_current_tab_left  <C-<pageup>>
       move_current_tab_right <C-<pagedown>>

Non-goals
- Do not add tab-reordering via control_menu; keeping control_menu generic avoids menu-specific logic in the router.
- Do not change existing Delete key behavior; we only add a configurable path in parallel.

Implementation Details
- File: pdf_viewer/main_widget.cpp
  - In MainWidget::handle_action_in_menu(std::wstring action):
      if (selector_widget && (action == L"delete" || action == L"remove" || action == L"del")) {
          selector_widget->handle_delete();
          return "";
      }
  - Add:
      void move_current_tab_left();
      void move_current_tab_right();

- File: pdf_viewer/main_widget.h
  - Declare the two new helper methods.

- File: pdf_viewer/document.h/.cpp
  - Add:
      void move_tab(int from, int to);
    Implementation: bounds-check, then std::rotate/erase+insert to reposition.

- File: pdf_viewer/input.cpp
  - Add Command classes move_current_tab_left/right similar to goto_next_tab commands, calling the MainWidget helpers.

- File: pdf_viewer/keys.config (optional docs comment)
  - Show example user binding for menu delete and tab moves.

Testing / Validation
- Manual:
  1) Open multiple tabs; open goto_tab; verify Delete still removes entry.
  2) Add in keys_user.config:
       [m] control_menu(delete) <C-d>
     Open goto_tab; press Ctrl+D → entry deletes.
  3) Bind and test tab moves:
       move_current_tab_left  <C-<pageup>>
       move_current_tab_right <C-<pagedown>>
     Observe order change via get_current_tabs_file_names() in status or by reopening goto_tab (list order should reflect vector order).

Risks and Mitigations
- Accidental deletes: behavior matches existing Delete key; this only exposes a configurable path.
- Tab move across windows: DocumentManager is shared; moving tabs only changes ordering, not window routing; MainWidget::handle_goto_tab continues to resolve correctly.

Safety/No‑op semantics for control_menu(delete)
- Only called when a selector is active: the router uses
  `dynamic_cast<BaseSelectorWidget*>(current_widget_stack.back())` (pdf_viewer/main_widget.cpp:9711),
  so if the active widget isn’t a selector, we do nothing.
- All selectors have `handle_delete()`: BaseSelectorWidget defines it (pdf_viewer/ui.cpp:1801) and it only
  calls `on_delete(...)` when there is a valid selection and mapping. Concrete list/table selectors then
  check if an `on_delete_function` exists before mutating the model (pdf_viewer/ui.h:358 for table,
  pdf_viewer/ui.h:543 for list). If no callback was provided by the menu, it’s a no‑op.
- Net effect: binding `[m] control_menu(delete)` is safe everywhere; it only has an effect in menus that
  intentionally provide an `on_delete` handler (e.g., goto_tab in pdf_viewer/main_widget.cpp:8339).

Docs
- Update README/docs to mention:
  - New control_menu(delete) action (menu-only binding via [m]).
  - New commands: move_current_tab_left / move_current_tab_right.

Future Work (Optional)
- Add control_menu actions for edit (call selector_widget->handle_edit()).
- Add menu-specific actions by tagging selector context if a future use case demands it (not required here).
