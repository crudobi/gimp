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

Design: in the per-tool action activation path, when the requested tool
is already the active tool of its group, advance the group's active
tool to the next tool in the group (iterate the group's children, wrap
around) instead of re-activating the same one. First press still selects
the group as today; only a repeat press cycles. Shift+key cycling is a
separate decision (Photoshop gates it behind a preference; GIMP has no
such preference yet).

## Fix 2 — Shift/Ctrl modifiers during a preliminary Free Select

Photoshop: closing a lasso commits it, so Shift+click immediately starts
an additive stroke. GIMP: Free Select is two-phase (draw polygon ->
preliminary outline -> Enter validates). While preliminary, Shift+clicks
are ignored entirely. Verified live 2026-11-15 in three phases, plus the
GIMP manual ("you can't use Shift, Ctrl yet. You have to hit Enter").

Code locations (gimp-3-2):
- app/tools/gimpfreeselecttool.c: button_press ~176, button_release,
  commit ~307 (calls gimp_free_select_tool_select when the polygon is
  not closed), operation captured at stroke start
  (priv->operation_at_start).
- app/tools/gimppolygonselecttool.c: button_press ~196, modifier_key
  ~317. The selection is not created until commit (Enter).

Design (narrow, preserves the two-phase model): in
gimp_free_select_tool_button_press, when a NORMAL press starts a new
stroke (priv->started is FALSE) and an uncommitted polygon exists and
the press carries Shift or Ctrl, commit the existing polygon first
with the modifier's operation, then let the new stroke start. Rect and
Ellipse select need nothing (no commit phase). The Mode=Add workaround
in Tool Options already proves the additive path works once a selection
exists.

## Not yet done

- Building GIMP 3.2 from source on kaw is untested (needs meson plus
  babl/gegl/GTK3 dev packages and the rest of the build deps). No C
  change lands without a build + GUI verification loop first.
- No GNOME GitLab account is configured; upstreaming would go through
  GitLab MRs. The GitHub mirror is the working fork for now.
- Ctrl+I (Photoshop: invert image) is intentionally unbound in the
  PhotoGIMP branch: GIMP 3.2's Colors > Invert is a dynamically
  registered GEGL filter action and the exact action name could not be
  confirmed from source.
