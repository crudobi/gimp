# Photoshop alignment — GIMP core fixes (working notes, drop before upstreaming)

Fork: github.com/crudobi/gimp (mirror of GNOME GitLab gimp-3-2).
Sibling fork: github.com/crudobi/PhotoGIMP, branch photoshop-alignment
(shortcutsrc fixes, commit f4d8659). This branch is the C side.

## Fix 1 — tap-to-cycle through tool groups

Photoshop: tools sharing one key cycle on repeated presses (B, B, B...
walks the brush group); Shift+key cycles when the preference is on.
GIMP 3.2: no cycling exists. Pressing a tool shortcut re-activates the
group's current tool. Verified live 2026-11-15 (B,B stays Paintbrush;
Shift+B is a no-op).

Implemented and GUI-verified live 2026-11-15: P walks Paintbrush,
Pencil, Airbrush, Ink, MyPaint Brush, then wraps to Paintbrush.
New gimp_tool_group_get_next_tool_info() (app/core/gimptoolgroup.c);
tools_select_cmd_callback (app/actions/tools-commands.c) cycles from
the currently active member whenever the requested tool's group is
already active, so any member's shortcut advances the group. A
shortcut from outside the group still selects that tool, as before.
Stock shortcuts: Paintbrush is P, Pencil N, Airbrush A, Paths B.

Code locations (gimp-3-2):
- Tool actions are built per tool in app/actions/tools-actions.c (~line 813
  loop over tool infos; entry.accelerator[0] = tool_info->menu_accel).
  The per-tool action callback assignment still needs confirming by
  reading that function.
- Groups are GimpToolGroup (app/core/gimptoolgroup.h/.c); the tool
  manager tracks the active group and its active tool
  (app/tools/tool_manager.c: tool_manager_set_active_tool_group ~737,
  tool_manager_group_active_tool_changed ~975, which calls
  gimp_context_set_tool).
- Activation itself goes through gimp_context_set_tool
  (tool_manager.c:299, 795, 899, 978, 999, 1020).

Design: in tools_select_cmd_callback, when the requested tool belongs
to the same group as the currently active tool, activate the next
visible tool in the group (iterate the group's children, wrap around)
instead of the requested one. First press from outside the group still
selects as today; only a repeat press from inside the group cycles.
Shift+key cycling is a separate decision (Photoshop gates it behind a
preference; GIMP has no such preference yet).

## Fix 2 — Shift/Ctrl modifiers during a preliminary Free Select

Photoshop: Shift+lasso adds to the selection, Ctrl+lasso subtracts,
and the modifier applies even when pressed mid-stroke. GIMP: Free
Select is two-phase (draw polygon -> preliminary outline -> Enter
validates), and the commit uses the operation captured at stroke
start (priv->operation_at_start). The bug: while the polygon widget
is active (from tool activation on),
gimp_polygon_select_tool_modifier_key() masked Shift/Ctrl out of the
state passed to the base selection-tool handler, so
options->operation never became Add/Subtract. The mode indicator
stayed Replace while Shift was held (verified live), and the commit
replaced instead of adding. Points DO land exactly where clicked
with Shift held (no angle-constraint snap, clicks not ignored); the
operation is the whole bug.

Code locations (gimp-3-2):
- app/tools/gimpfreeselecttool.c: button_press ~176, button_release,
  commit ~307 (calls gimp_free_select_tool_select when the polygon is
  not closed), operation captured at stroke start
  (priv->operation_at_start).
- app/tools/gimppolygonselecttool.c: button_press ~196, modifier_key
  ~317. The selection is not created until commit (Enter).

Implemented and GUI-verified live 2026-11-15 (Quick Mask as ground
truth: both diamonds selected after a mid-stroke Shift):
- gimppolygonselecttool.c: the masking is removed; the base handler
  now sees the real modifiers and sets options->operation live. The
  widget already received the unmasked state via hover_modifier, so
  its own Shift handling is unaffected.
- gimpfreeselecttool.c: new modifier_key override latches
  options->operation into operation_at_start whenever Shift/Ctrl is
  held during an open stroke, so a mid-stroke Shift makes the commit
  Add. Releasing before commit keeps the latched operation; starting
  with Shift held works as before. Alt is excluded (GIMP pretends no
  modifier while Alt is held).

Superseded (do not revive): a commit-on-Shift-click hook in
button_press. It fired on EVERY Shift+click once n_points > 2, so it
committed the new polygon after just 3 points (proven live: the 4th
Shift+click committed the 3-point triangle instead of adding the 4th
vertex). A Shift+click cannot distinguish "start a new stroke" from
"continue this stroke", so committing on it is wrong.

Note on verification: marching ants do not render visibly in the Xvfb
screenshots, so ant absence proves nothing here. Judge selections by
Quick Mask (Shift+Q) or the Selection Editor, not by ants.

## Build (done 2026-11-15, on kaw)

- GIMP 3.2.9 builds from source on kaw (meson + ninja; babl 0.1.129
  and gegl 0.4.73 built from GNOME GitLab master into
  ~/gimp-deps, because the system babl/gegl are too old; 58 apt
  -dev packages + libappstream-dev + xsltproc). Scripts in /tmp on
  kaw: kaw-gimp-builddeps.sh, kaw-gimp-babl-gegl.sh,
  kaw-gimp-meson-setup.sh, kaw-gimp-build.sh, kaw-gimp-install.sh.
  Prefix ~/gimp-deps; the binary has no rpath, run it with
  LD_LIBRARY_PATH set. GUI verification under Xvfb :99 with xdotool
  and a scratch profile (never the real PhotoGIMP profile).
- Known gap: the dev install is missing the Script-Fu interpreter
  (gimp-script-fu-interpreter-3.0 not found at startup), so
  Script-Fu does not work in the dev build. Non-fatal for tool work.
- Marching ants do not render visibly in Xvfb screenshots; judge
  selections by Quick Mask or the Selection Editor.

## Not yet done

- No GNOME GitLab account is configured; upstreaming would go through
  GitLab MRs. The GitHub mirror is the working fork for now.
- Ctrl+I (Photoshop: invert image) is intentionally unbound in the
  PhotoGIMP branch: GIMP 3.2's Colors > Invert is a dynamically
  registered GEGL filter action and the exact action name could not be
  confirmed from source.
