# Professional GUI Patterns

Production-grade UI patterns extracted from COT (Community Online Tools), VPP Admin Tools, DabsFramework, and Expansion. These solve architecture problems that basic widget knowledge cannot: managing 12+ admin panels, drawing on-screen overlays, formatting rich text, and building extensible window systems.

---

## Module-Form-Window Architecture (COT)

The standard architecture for scalable admin panels. Used by COT to manage 12+ tools (ESP, Player Manager, Teleport, Object Spawner, etc.) without code duplication.

### Three-Layer Separation

1. **JMRenderableModuleBase** — Metadata layer. Declares title, icon, layout path, permissions. Manages CF_Window lifecycle. No UI logic.
2. **JMFormBase** — UI logic layer. Extends `ScriptedWidgetEventHandler`. Receives widget events, builds UI elements, talks to the module for data.
3. **CF_Window** — Windowing container from CF. Handles drag, resize, close chrome.

### Module Declaration

```c
class JMExampleModule: JMRenderableModuleBase
{
    void JMExampleModule()
    {
        GetPermissionsManager().RegisterPermission("Admin.Example.View");
        GetPermissionsManager().RegisterPermission("Admin.Example.Button");
    }

    override bool HasAccess()
    {
        return GetPermissionsManager().HasPermission("Admin.Example.View");
    }

    override string GetLayoutRoot()
    {
        return "JM/COT/GUI/layouts/Example_form.layout";
    }

    override string GetTitle() { return "Example Module"; }
    override string GetIconName() { return "E"; }
    override bool ImageIsIcon() { return false; }
}
```

### Module Registration

```c
modded class JMModuleConstructor
{
    override void RegisterModules(out TTypenameArray modules)
    {
        super.RegisterModules(modules);
        modules.Insert(JMPlayerModule);
        modules.Insert(JMObjectSpawnerModule);
        modules.Insert(JMESPModule);
        modules.Insert(JMTeleportModule);
        // One line per tool — no existing code changes needed
    }
}
```

### Form Binding

```c
class JMExampleForm: JMFormBase
{
    protected JMExampleModule m_Module;

    protected override bool SetModule(JMRenderableModuleBase mdl)
    {
        return Class.CastTo(m_Module, mdl);
    }

    override void OnInit()
    {
        // Build UI elements using UIActionManager
    }
}
```

### Window Show/Hide

```c
void Show()
{
    if (HasAccess())
    {
        m_Window = new CF_Window();
        Widget widgets = m_Window.CreateWidgets(GetLayoutRoot());
        widgets.GetScript(m_Form);
        m_Form.Init(m_Window, this);
    }
}
```

**Key benefit:** Adding a new admin tool = 1 Module class + 1 Form class + 1 layout file + 1 line in constructor. Zero changes to existing code.

---

## UIActionManager Factory Pattern (COT)

COT builds complex forms programmatically using a factory class instead of monolithic layout files:

```c
override void OnInit()
{
    m_Scroller = UIActionManager.CreateScroller(layoutRoot.FindAnyWidget("panel"));
    Widget actions = m_Scroller.GetContentWidget();

    // Grid layout: 8 rows, 1 column
    m_PanelAlpha = UIActionManager.CreateGridSpacer(actions, 8, 1);

    // Factory methods for each widget type
    m_Text = UIActionManager.CreateText(m_PanelAlpha, "Label", "Value");
    m_EditableText = UIActionManager.CreateEditableText(
        m_PanelAlpha, "Name:", this, "OnChange_EditableText"
    );
    m_Slider = UIActionManager.CreateSlider(
        m_PanelAlpha, "Speed:", 0, 100, this, "OnChange_Slider"
    );
    m_Checkbox = UIActionManager.CreateCheckbox(
        m_PanelAlpha, "Enable Feature", this, "OnClick_Checkbox"
    );
    m_Button = UIActionManager.CreateButton(
        m_PanelAlpha, "Execute", this, "OnClick_Button"
    );

    // Sub-grid for side-by-side buttons
    Widget gridButtons = UIActionManager.CreateGridSpacer(m_PanelAlpha, 1, 2);
    UIActionManager.CreateButton(gridButtons, "Left", this, "OnClick_Left");
    UIActionManager.CreateButton(gridButtons, "Right", this, "OnClick_Right");
}
```

Each `UIAction*` widget type has its own prefab layout file. Benefits:
- Consistent sizing and spacing across all panels
- No layout file duplication
- New action types added once, used everywhere
- Callback routing via string method names (`this, "OnChange_Slider"`)

---

## VPP Sub-Window System

VPP builds a mini window manager inside DayZ with draggable, resizable panels:

### Toolbar Registration (Extensible)

```c
class VPPAdminHud extends VPPScriptedMenu
{
    private ref array<ref VPPButtonProperties> m_DefinedButtons;

    void VPPAdminHud()
    {
        InsertButton("MenuPlayerManager", "Player Manager",
            "set:dayz_gui_vpp image:vpp_icon_players", "#VSTR_TOOLTIP_PLAYER");
        InsertButton("MenuItemManager", "Items Spawner",
            "set:dayz_gui_vpp image:vpp_icon_item_manager", "#VSTR_TOOLTIP_ITEMS");
        DefineButtons();

        // Verify permissions with server via RPC
        array<string> perms = new array<string>();
        for (int i = 0; i < m_DefinedButtons.Count(); i++)
            perms.Insert(m_DefinedButtons[i].param1);
        GetRPCManager().VSendRPC("RPC_PermitManager",
            "VerifyButtonsPermission", new Param1<ref array<string>>(perms), true);
    }
}
```

External mods override `DefineButtons()` to add their own toolbar buttons.

### Draggable Sub-Menu Window

```c
class AdminHudSubMenu: ScriptedWidgetEventHandler
{
    protected Widget M_SUB_WIDGET;
    protected Widget m_TitlePanel;

    void ShowSubMenu()
    {
        m_IsVisible = true;
        M_SUB_WIDGET.Show(true);
        VPPAdminHud rootHud = VPPAdminHud.Cast(
            GetVPPUIManager().GetMenuByType(VPPAdminHud));
        rootHud.SetWindowPriorty(this);  // Bring to front
        OnMenuShow();
    }

    override bool OnDrag(Widget w, int x, int y)
    {
        if (w == m_TitlePanel)
        {
            M_SUB_WIDGET.GetPos(m_posX, m_posY);
            m_posX = x - m_posX;
            m_posY = y - m_posY;
            return false;
        }
        return true;
    }

    override bool OnDragging(Widget w, int x, int y, Widget reciever)
    {
        if (w == m_TitlePanel)
        {
            SetWindowPos(x - m_posX, y - m_posY);
            return false;
        }
        return true;
    }
}
```

---

## Confirmation Dialogs with Callback Routing

### COT Pattern (Named Callbacks)

```c
// Create confirmation
CreateConfirmation_Two(
    JMConfirmationType.INFO,
    "Are you sure?",
    "This will kick the player.",
    "#STR_COT_GENERIC_YES", "OnConfirmKick",
    "#STR_COT_GENERIC_NO", ""
);

// Callback is invoked via CallByName
void OnConfirmKick(JMConfirmation confirmation)
{
    // User confirmed — proceed with kick
    KickPlayer(m_SelectedPlayer);
}
```

### VPP Pattern (Enum-Driven)

```c
enum DIAGTYPE
{
    DIAG_YESNO,
    DIAG_YESNOCANCEL,
    DIAG_OK,
    DIAG_OK_CANCEL_INPUT
}

class VPPDialogBox extends ScriptedWidgetEventHandler
{
    private Class m_CallBackClass;
    private string m_CbFunc = "OnDiagResult";

    void InitDiagBox(int diagType, string title, string content,
                     Class callBackClass, string cbFunc = string.Empty)
    {
        m_CallBackClass = callBackClass;
        if (cbFunc != string.Empty) m_CbFunc = cbFunc;
        // Show/hide buttons based on diagType
    }

    private void OnOutCome(int result)
    {
        GetGame().GameScript.CallFunction(m_CallBackClass, m_CbFunc, null, result);
        delete this;
    }
}
```

Both approaches route dialog results through named callbacks, enabling chained confirmations without hardcoded flow.

---

## RichTextWidget & HtmlWidget

For formatted text with inline images, headings, and line breaks.

### Supported Tags

```
<image set="IMAGESET_NAME" name="IMAGE_NAME" />
<image set="IMAGESET_NAME" name="IMAGE_NAME" scale="1.5" />
</br>
<h scale="0.8">Heading text</h>
```

### Common ImageSets

| ImageSet | Contents |
|----------|----------|
| `dayz_gui` | General UI icons (pin, notifications) |
| `dayz_inventory` | Inventory slot icons |
| `xbox_buttons` | Xbox controller buttons (A, B, X, Y) |
| `playstation_buttons` | PlayStation controller buttons |

### Usage

```c
RichTextWidget m_Label;
m_Label = RichTextWidget.Cast(root.FindAnyWidget("MyRichLabel"));

// Inline image + text
m_Label.SetText("<image set=\"dayz_gui\" name=\"icon_pin\" /> Welcome!");

// Controller button icons (vanilla helper)
string icon = InputUtils.GetRichtextButtonIconFromInputAction(
    "UAUISelect",                   // input action name
    "#menu_select",                 // localized label
    EUAINPUT_DEVICE_CONTROLLER,
    InputUtils.ICON_SCALE_TOOLBAR   // 1.81 scale
);
m_Label.SetText(icon);

// Scrollable content
float totalHeight = m_Label.GetContentHeight();
m_Label.SetContentOffset(pageOffset, true);  // snapToLine

// Text elision
m_Label.ElideText(0, maxWidth, "...");

// Line visibility control
m_Label.SetLinesVisibility(5, m_Label.GetNumLines() - 1, false);
```

### HtmlWidget (Extends RichTextWidget)

```c
HtmlWidget content;
Class.CastTo(content, layoutRoot.FindAnyWidget("HtmlWidget"));
content.LoadFile(book.ConfigGetString("file"));  // Load .html text file
```

### When to Use Which

| Feature | TextWidget | RichTextWidget |
|---------|-----------|---------------|
| Inline images | No | Yes |
| Heading tags | No | Yes |
| Line breaks `</br>` | No (use `\n`) | Yes |
| Content scrolling | No | Yes |
| Performance | Faster | Slower (tag parsing) |

Use `TextWidget` for simple labels. Use `RichTextWidget` only when you need inline images or formatted content.

---

## CanvasWidget Drawing

Immediate-mode 2D drawing. The ONLY way to draw lines, shapes, and HUD overlays on screen.

### The Entire API

```c
class CanvasWidget extends Widget
{
    proto native void DrawLine(float x1, float y1, float x2, float y2,
                               float width, int color);
    proto native void Clear();
}
```

All complex shapes must be built from line segments.

### Layout Setup

```
CanvasWidgetClass MyCanvas {
    ignorepointer 1     // Don't block mouse input
    position 0 0
    size 1 1             // Fill parent
    hexactpos 1
    vexactpos 1
    hexactsize 0
    vexactsize 0
}
```

### Drawing Primitives

```c
// Line (ARGB color format)
m_Canvas.DrawLine(10, 50, 200, 50, 2, ARGB(255, 255, 0, 0));

// Rectangle (from lines)
void DrawRectangle(CanvasWidget canvas, float x, float y,
                   float w, float h, float lineWidth, int color)
{
    canvas.DrawLine(x, y, x + w, y, lineWidth, color);
    canvas.DrawLine(x + w, y, x + w, y + h, lineWidth, color);
    canvas.DrawLine(x + w, y + h, x, y + h, lineWidth, color);
    canvas.DrawLine(x, y + h, x, y, lineWidth, color);
}

// Circle (from segments — 36 = smooth, 12 = cheap)
void DrawCircle(float cx, float cy, float radius,
                int lineWidth, int color, int segments)
{
    float segAngle = 360.0 / segments;
    for (int i = 0; i < segments; i++)
    {
        float a1 = i * segAngle * Math.DEG2RAD;
        float a2 = (i + 1) * segAngle * Math.DEG2RAD;
        m_Canvas.DrawLine(
            cx + radius * Math.Cos(a1), cy + radius * Math.Sin(a1),
            cx + radius * Math.Cos(a2), cy + radius * Math.Sin(a2),
            lineWidth, color
        );
    }
}
```

### Per-Frame Redraw Pattern

```c
override void Update(float timeslice)
{
    super.Update(timeslice);
    m_Canvas.Clear();  // MUST clear every frame
    DrawOverlay();     // Redraw everything
}
```

### ESP Overlay Pattern (World-to-Screen)

```c
// Convert 3D world position to 2D screen coordinates
vector TransformToScreenPos(vector worldPos, out bool isInBounds)
{
    vector screenPos = GetGame().GetScreenPosRelative(worldPos);

    isInBounds = screenPos[0] >= 0 && screenPos[0] <= 1
              && screenPos[1] >= 0 && screenPos[1] <= 1
              && screenPos[2] >= 0;  // In front of camera

    float parentW, parentH;
    m_Canvas.GetScreenSize(parentW, parentH);
    screenPos[0] = screenPos[0] * parentW;
    screenPos[1] = screenPos[1] * parentH;

    return screenPos;
}

// Draw line between two world positions
void DrawWorldLine(vector from, vector to, int width, int color)
{
    bool inA, inB;
    from = TransformToScreenPos(from, inA);
    to = TransformToScreenPos(to, inB);
    if (!inA || !inB) return;
    m_Canvas.DrawLine(from[0], from[1], to[0], to[1], width, color);
}
```

### Performance Tips

- `Clear()` and redraw every frame
- Minimize line count (fewer segments for distant objects)
- Check screen bounds before drawing (skip off-screen)
- Always set `ignorepointer 1` on canvas overlays
- Use ONE full-screen canvas for all overlay drawing

---

## MapWidget

Displays the DayZ terrain map with markers and coordinate conversion.

```c
MapWidget m_Map = MapWidget.Cast(layoutRoot.FindAnyWidget("Map"));

// Add marker
m_Map.AddUserMark(worldPos, "Base", ARGB(255, 255, 0, 0), "set:dayz_gui image:icon_pin");

// Clear markers
m_Map.ClearUserMarks();

// Pan and zoom
m_Map.SetMapPos(worldPos);
m_Map.SetScale(0.5);

// Coordinate conversion
vector screenPos = m_Map.MapToScreen(worldPos);
vector worldPos = m_Map.ScreenToMap(Vector(screenX, screenY, 0));
```

World coordinates: `x` = east/west, `y` = altitude, `z` = north/south. Chernarus range: ~0-15360 on x and z.

---

## Focus Management (Critical for Input)

Every UI panel that captures mouse/keyboard input MUST manage game focus correctly:

```c
void OpenPanel()
{
    m_Root.Show(true);
    GetGame().GetInput().ChangeGameFocus(1);       // Capture input
    GetGame().GetUIManager().ShowUICursor(true);    // Show cursor
}

void ClosePanel()
{
    m_Root.Show(false);
    GetGame().GetInput().ChangeGameFocus(-1);       // Release input
    GetGame().GetUIManager().ShowUICursor(false);   // Hide cursor
}
```

**CRITICAL:** Every `ChangeGameFocus(1)` MUST have a matching `ChangeGameFocus(-1)`. Imbalanced calls leave the player unable to move. Ensure cleanup runs even if the UI is force-closed or the player disconnects.

---

## Layout File Format

Layout files (`.layout`) use a brace-delimited format, NOT XML. Widget types use `Class` suffix.

### Basic Structure
```
TextWidgetClass MyLabel {
    position 0.1 0.05
    size 0.3 0.04
    hexactpos 0
    vexactpos 0
    hexactsize 0
    vexactsize 0
    text "Hello World"
    font "gui/fonts/MetronBook24"
    "exact text" 1
    "exact text size" 14
    visible 1
    color 1 1 1 1

    ButtonWidgetClass ChildButton {
        position 0 0.8
        size 0.2 0.05
        text "Click Me"
        scriptclass "MyButtonHandler"
    }
}
```

### Key Rules
- Widget types use `Class` suffix: `TextWidgetClass`, `ButtonWidgetClass`, `ImageWidgetClass`
- `key value` pairs (no `=` sign)
- Multi-word attribute names in quotes: `"exact text size" 14`
- Children nested in `{ }` blocks
- Color format: `color r g b a` with four floats (0.0-1.0), NOT ARGB integer
- `visible 0` to start hidden
- `priority` for z-ordering (integer)

### scriptclass Binding
Binds a layout widget to an Enforce Script class:
```
FrameWidgetClass MyPanel {
    scriptclass "MyPanelController"
    ...
}
```
The class MUST inherit from `Managed` and implement `OnWidgetScriptInit(Widget w)`.
If the class doesn't inherit Managed or has a constructor error, the handler is silently null.

### Gotchas
- `scriptclass` names must be globally unique across all mods
- 500+ widgets in one layout cause measurable frame drops
- Some widget types ignore alpha in `color` — need `inheritalpha 1` on parent
- PanelWidgetClass has NO `PanelWidget` script class — must cast to base `Widget`

---

## Sizing & Positioning System

### Dual Coordinate Mode
Four flags independently control proportional vs pixel per axis:

| Flag | 0 (default) | 1 |
|------|-------------|---|
| `hexactpos` | Proportional X position (0.0-1.0) | Pixel X position |
| `vexactpos` | Proportional Y position (0.0-1.0) | Pixel Y position |
| `hexactsize` | Proportional width (0.0-1.0) | Pixel width |
| `vexactsize` | Proportional height (0.0-1.0) | Pixel height |

### Alignment
```
halign left_ref      // left_ref, center_ref, right_ref
valign center_ref    // top_ref, center_ref, bottom_ref
```
With `center_ref` + pixel pos `0 0` = centered in parent.

### Z-Order Priority Ranges
| Range | Use |
|-------|-----|
| 0-5 | Background panels |
| 10-50 | Normal UI elements |
| 50-100 | Overlays |
| 100-200 | Notifications |
| 998-999 | Modals/Dialogs |

Priority only affects siblings within the same parent. Children always render on top of parent.

### Common Sizing Patterns
```c
// Full-screen overlay (proportional)
widget.SetPos(0, 0);
widget.SetSize(1, 1);

// Centered dialog (pixel size, center-aligned)
// In layout: halign center_ref, valign center_ref, hexactsize 1, vexactsize 1
widget.SetSize(400, 300);
widget.SetPos(0, 0);  // Centered due to alignment refs
```

### CRITICAL
- **No negative size values** — causes undefined behavior, invisible widgets, or UI crashes
- `SetScreenPos`/`SetScreenSize` must be called after widget is attached to parent
- `scaled` attribute respects DayZ's HUD Size setting (only pixel-mode dimensions)

---

## Container Widgets

### WrapSpacerWidget
Auto-wraps children horizontally/vertically.
- **Must call `Update()` after `CreateWidgets()` or `Unlink()`** — layout does NOT auto-recalculate
- WrapSpacer inside WrapSpacer with `Size To Content` on both can cause infinite layout loops

### GridSpacerWidget
Arranges children in a fixed grid.
- **Overrides children's size attributes entirely** — setting size on a grid child has no effect
- Children fill cells in order of creation

### ScrollWidget
Scrollable content area.
```c
ScrollWidget scroll;
scroll.VScrollToPos(0);                  // Scroll to top
float pos = scroll.GetVScrollPos();      // Current scroll position
float height = scroll.GetContentHeight(); // Total content height
scroll.VScrollStep(1);                   // Scroll one step
```
- Always set `clipchildren 1` on ScrollWidget
- After adding children, may need to defer `VScrollToPos()` by one frame via `CallLater`
- 50+ children cause frame drops — use widget pooling for large lists

---

## Complete Event Handler Reference

### ScriptedWidgetEventHandler Methods

| Method | Parameters | When Fired |
|--------|-----------|------------|
| `OnClick` | `Widget w, int x, int y, int button` | Button click (0=left, 1=right, 2=middle) |
| `OnDoubleClick` | `Widget w, int x, int y, int button` | Double-click |
| `OnChange` | `Widget w, int x, int y, bool finished` | Value changed (finished=true on Enter) |
| `OnFocus` | `Widget w, int x, int y` | Widget gains focus |
| `OnFocusLost` | `Widget w, int x, int y` | Widget loses focus |
| `OnMouseEnter` | `Widget w, int x, int y` | Mouse enters widget |
| `OnMouseLeave` | `Widget w, Widget enterW, int x, int y` | Mouse leaves (enterW = new target) |
| `OnMouseWheel` | `Widget w, int x, int y, int wheel` | Scroll (positive=up, negative=down) |
| `OnMouseButtonDown` | `Widget w, int x, int y, int button` | Mouse button pressed |
| `OnMouseButtonUp` | `Widget w, int x, int y, int button` | Mouse button released |
| `OnKeyDown` | `Widget w, int x, int y, int key` | Keyboard key pressed |
| `OnKeyUp` | `Widget w, int x, int y, int key` | Keyboard key released |
| `OnDrag` | `Widget w, int x, int y` | Drag started |
| `OnDragging` | `Widget w, int x, int y, Widget reciever` | During drag |
| `OnDrop` | `Widget w, int x, int y` | Drag ended |
| `OnDropReceived` | `Widget w, int x, int y, Widget receiver` | Item dropped on this widget |
| `OnResize` | `Widget w, int x, int y` | Widget resized |
| `OnUpdate` | `Widget w` | Per-frame update |
| `OnItemSelected` | `Widget w, int x, int y, int row, int col, int oldRow, int oldCol` | Listbox selection |

### Return Value Semantics
- `return true` = event consumed, stops propagation to parent
- `return false` = event passed to parent widget handler

### WidgetEventHandler Singleton (Alternative)
Per-widget, per-event named callbacks — supports multiple handlers on one widget:
```c
WidgetEventHandler.GetInstance().RegisterOnClick(myButton, this, "OnButtonClicked");
WidgetEventHandler.GetInstance().RegisterOnMouseEnter(myButton, this, "OnButtonHover");

// Cleanup
WidgetEventHandler.GetInstance().UnregisterWidget(myButton);
```

**Key difference:** `SetHandler()` allows only ONE handler per widget. `WidgetEventHandler.RegisterOnClick()` is additive — multiple mods can register on the same widget.

### Gotchas
- `OnClick` fires reliably ONLY on `ButtonWidget`. For other types use `OnMouseButtonDown`/`OnMouseButtonUp`
- `SetHandler()` replaces old handler silently — old handler's resources leak
- `TextWidget` does NOT have `GetText()`. Only `EditBoxWidget` and `ButtonWidget` have it. Store text in a variable when calling `SetText()`

---

## UIScriptedMenu System

Engine base class for menus with automatic focus management:

```c
class MyMenu extends UIScriptedMenu
{
    override Widget Init()
    {
        layoutRoot = GetGame().GetWorkspace().CreateWidgets("MyMod/gui/layouts/menu.layout");
        return layoutRoot;
    }

    override void OnShow()
    {
        super.OnShow();
        // Auto-called, focus is captured
    }

    override void OnHide()
    {
        super.OnHide();
        // Auto-called, focus is released
    }

    override void Update(float timeslice)
    {
        super.Update(timeslice);
        // ESC to close
        if (GetUApi().GetInputByID(UAUIBack).LocalPress())
            Close();
    }
}

// Show/hide
GetGame().GetUIManager().ShowScriptedMenu(new MyMenu(), null);
menu.Close();
```

---

## ShowDialog() — Native Engine Dialogs

Simple Yes/No/OK prompts:
```c
g_Game.GetUIManager().ShowDialog(
    "Confirm",                    // Caption
    "Are you sure?",              // Text
    DIALOG_ID_CONFIRM,            // ID for result routing
    DBT_YESNO,                    // Button type: DBT_OK, DBT_YESNO, DBT_YESNOCANCEL
    DBB_YES,                      // Default button: DBB_NONE, DBB_OK, DBB_YES, DBB_NO, DBB_CANCEL
    DMT_QUESTION,                 // Icon: DMT_NONE, DMT_INFO, DMT_WARNING, DMT_QUESTION, DMT_EXCLAMATION
    this                          // Handler (receives OnModalResult)
);

// Handle result
override bool OnModalResult(Widget w, int x, int y, int code, int result)
{
    if (code == DIALOG_ID_CONFIRM && result == DBB_YES)
        DoConfirmedAction();
    return true;
}
```

---

## Interactive Widget APIs

### CheckBoxWidget
```c
CheckBoxWidget cb;
bool checked = cb.IsChecked();
cb.SetChecked(true);
// OnChange fires when toggled
```

### EditBoxWidget
```c
EditBoxWidget edit;
string text = edit.GetText();
edit.SetText("Hello");
// OnChange: finished=true means Enter was pressed
```

### XComboBoxWidget (Dropdown)
```c
XComboBoxWidget combo;
combo.AddItem("Option A");
combo.AddItem("Option B");
combo.SetCurrentItem(0);
int selected = combo.GetCurrentItem();
combo.ClearAll();
// OnChange fires on selection
```

### TextListboxWidget
```c
TextListboxWidget list;
list.AddItem("Row 1", null, 0);
list.AddItem("Row 2", null, 0);
int row = list.GetSelectedRow();
list.SelectRow(0);
list.RemoveRow(1);
list.ClearItems();
// OnItemSelected fires on selection
```

### SliderWidget
```c
SliderWidget slider;
slider.SetMinMax(0, 100);
slider.SetCurrent(50);
float val = slider.GetCurrent();
// Layout: "fill in" 1 for filled track, "listen to input" 1 for mouse
```

### ProgressBarWidget
```c
ProgressBarWidget bar;
bar.SetCurrent(75.0);  // 0-100 range
```

---

## 3D Preview Widgets

### ItemPreviewWidget
```c
ItemPreviewWidget preview;
preview.SetItem(item);
preview.SetModelOrientation(Vector(0, 0, 0));
preview.SetModelPosition(Vector(0, 0, 0.5));
// View indices: 0=default, 1=alternate
preview.SetView(0);
```

### PlayerPreviewWidget
```c
PlayerPreviewWidget playerPreview;
playerPreview.SetPlayer(player);
playerPreview.UpdateItemInHands(item);
playerPreview.Refresh();
playerPreview.SetModelOrientation(Vector(0, 0, 0));
// GetDummyPlayer() returns internal dummy — do NOT destroy it
```

---

## Styles & Fonts

### Built-in Styles
`blank`, `Empty`, `Default`, `Colorable`, `DayZNormal`, `MenuDefault`, `OutlineFilled`, `Outline_1px_BlackBackground`

### Built-in Fonts
| Font Path | Notes |
|-----------|-------|
| `gui/fonts/Metron` | Default sans-serif |
| `gui/fonts/Metron28` | Larger variant |
| `gui/fonts/Metron-Bold` | Bold weight |
| `gui/fonts/sdf_MetronBook24` | SDF — renders crisp at any size (best for variable-scale) |

### Text Sizing
- Proportional (default): Font scales with widget height
- Exact pixel: `"exact text" 1` + `"exact text size" 14`
- Auto-size widget to text: `"size to text h" 1` / `"size to text v" 1`

### ImageSet Registration
Custom imagesets must be registered in config.cpp:
```cpp
class defs
{
    class imageSets
    {
        files[] = { "MyMod/gui/imagesets/my_icons.imageset" };
    };
};
```
Without registration, images fail silently (blank widgets).

Reference syntax: `set:SETNAME image:IMAGENAME`

### Color Gotcha
`SetColor()` tints the widget's existing visual. On styled widgets, it MULTIPLIES with the style's base color — unexpected results.
