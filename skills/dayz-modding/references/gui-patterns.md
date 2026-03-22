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
