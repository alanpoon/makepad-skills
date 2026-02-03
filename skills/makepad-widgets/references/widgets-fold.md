# FoldButton and FoldHeader Widgets Reference

Makepad's built-in widgets for creating collapsible/accordion UI patterns.

## FoldButton

A triangular fold indicator button that animates between open and closed states.

### Import

```rust
use makepad_widgets::fold_button::FoldButtonAction;
```

### Live Design

```rust
live_design! {
    use link::widgets::*;

    MyFoldButton = <FoldButton> {
        // Default size
        width: 12, height: 12

        // Optional: customize the triangle appearance
        draw_bg: {
            // Shader customization here
        }

        // Animator states: active.on (open) and active.off (closed)
        animator: {
            active = {
                default: off
                off = {
                    from: { all: Forward { duration: 0.2 } }
                    apply: { draw_bg: { active: 0.0 } }
                }
                on = {
                    from: { all: Forward { duration: 0.2 } }
                    apply: { draw_bg: { active: 1.0 } }
                }
            }
        }
    }
}
```

### Actions

FoldButton emits these actions via `FoldButtonAction`:

```rust
use makepad_widgets::fold_button::FoldButtonAction;

// In handle_actions or event handling:
match action.cast() {
    FoldButtonAction::Opening => {
        // Button transitioning to open state
    }
    FoldButtonAction::Closing => {
        // Button transitioning to closed state
    }
    FoldButtonAction::Animating(value) => {
        // Animation in progress, value is 0.0 to 1.0
    }
    _ => {}
}
```

### Helper Methods

```rust
impl FoldButtonRef {
    // Check if button is opening
    pub fn opening(&self, actions: &Actions) -> bool;

    // Check if button is closing
    pub fn closing(&self, actions: &Actions) -> bool;

    // Get current animation value
    pub fn animating(&self, actions: &Actions) -> Option<f64>;
}
```

---

## FoldHeader

A container widget with a header (always visible) and body (collapsible) section.

### Import

```rust
use makepad_widgets::fold_header::FoldHeaderWidgetRefExt;
```

### Basic Structure

```rust
live_design! {
    use link::widgets::*;

    MyFoldHeader = <FoldHeader> {
        // Header: Always visible, controls fold state
        header: <View> {
            width: Fill, height: 50
            flow: Right
            align: { y: 0.5 }
            padding: { left: 10, right: 10 }

            // REQUIRED: Include a FoldButton to control the fold state
            fold_button = <FoldButton> {}

            // Add any header content
            title = <Label> { text: "Section Title" }
        }

        // Body: Collapsible content
        body: <View> {
            width: Fill, height: Fit
            flow: Down
            padding: 10

            // Add collapsible content here
            content = <Label> { text: "Hidden content" }
        }
    }
}
```

### Key Points

1. **No prefix needed**: Access nested widgets directly without "header" or "body" prefixes
   ```rust
   // Correct
   fold_header.label(ids!(title)).set_text(cx, "New Title");

   // NOT needed
   // fold_header.label(ids!(header.title))  // Don't do this
   ```

2. **FoldButton is required**: The header must contain a `FoldButton` (or `FoldButtonWithText`) for the fold behavior to work

3. **Body height**: Use `height: Fit` for the body to automatically size to content

### Accessing FoldHeader in Code

```rust
use makepad_widgets::fold_header::FoldHeaderWidgetRefExt;

impl Widget for MyWidget {
    fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) {
        // Get FoldHeader reference
        let fold_header = self.view.fold_header(ids!(my_fold_header));

        // Access nested widgets (no header/body prefix needed)
        fold_header.label(ids!(title)).set_text(cx, "Updated Title");
        fold_header.button(ids!(action_btn)).set_text(cx, "Click Me");
    }
}
```

### Converting WidgetRef to FoldHeader

```rust
// From a WidgetRef (e.g., from portal list item)
let widget_ref = list.item(cx, item_id, live_id!(MyFoldHeader));
let fold_header = widget_ref.as_fold_header();
```

---

## FoldButtonWithText (Custom Extension)

A custom widget that extends FoldButton with dynamic text labels.

### Implementation

```rust
use makepad_widgets::*;
use makepad_widgets::widget::WidgetActionData;
use makepad_widgets::fold_button::FoldButtonAction;  // IMPORTANT: Use existing action type

#[derive(Live, Widget)]
pub struct FoldButtonWithText {
    #[animator] animator: Animator,
    #[redraw] #[live] draw_bg: DrawQuad,
    #[redraw] #[live] draw_text: DrawText,
    #[walk] walk: Walk,
    #[layout] layout: Layout,
    #[live] active: f64,
    #[live] triangle_size: f64,
    #[live] open_text: ArcStringMut,   // Text when collapsed (shown to indicate "click to open")
    #[live] close_text: ArcStringMut,  // Text when expanded (shown to indicate "click to close")
    #[action_data] #[rust] action_data: WidgetActionData,
}

impl Widget for FoldButtonWithText {
    fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) {
        let uid = self.widget_uid();
        let res = self.animator_handle_event(cx, event);

        if res.is_animating() {
            if self.animator.is_track_animating(cx, ids!(active)) {
                let mut value = [0.0];
                self.draw_bg.get_instance(cx, ids!(active), &mut value);
                cx.widget_action(uid, &scope.path, FoldButtonAction::Animating(value[0] as f64))
            }
            if res.must_redraw() {
                self.draw_bg.redraw(cx);
            }
        }

        match event.hits(cx, self.draw_bg.area()) {
            Hit::FingerDown(_fe) => {
                if self.animator_in_state(cx, ids!(active.on)) {
                    self.animator_play(cx, ids!(active.off));
                    cx.widget_action(uid, &scope.path, FoldButtonAction::Closing)
                } else {
                    self.animator_play(cx, ids!(active.on));
                    cx.widget_action(uid, &scope.path, FoldButtonAction::Opening)
                }
                self.animator_play(cx, ids!(hover.on));
            },
            Hit::FingerHoverIn(_) => {
                cx.set_cursor(MouseCursor::Hand);
                self.animator_play(cx, ids!(hover.on));
            }
            Hit::FingerHoverOut(_) => {
                self.animator_play(cx, ids!(hover.off));
            }
            Hit::FingerUp(fe) => {
                if fe.is_over && fe.device.has_hovers() {
                    self.animator_play(cx, ids!(hover.on));
                } else {
                    self.animator_play(cx, ids!(hover.off));
                }
            }
            _ => ()
        }
    }

    fn draw_walk(&mut self, cx: &mut Cx2d, _scope: &mut Scope, walk: Walk) -> DrawStep {
        self.draw_bg.begin(cx, walk, self.layout);

        // Dynamically select text based on state
        let text = if self.active > 0.5 {
            self.close_text.as_ref()  // Expanded state
        } else {
            self.open_text.as_ref()   // Collapsed state
        };

        let label_walk = walk.with_margin_left(self.triangle_size * 2.0 + 10.0);
        self.draw_text.draw_walk(cx, label_walk, Align::default(), text);
        self.draw_bg.end(cx);
        DrawStep::done()
    }
}

impl FoldButtonWithText {
    pub fn opening(&self, actions: &Actions) -> bool {
        if let Some(item) = actions.find_widget_action(self.widget_uid()) {
            if let FoldButtonAction::Opening = item.cast() {
                return true
            }
        }
        false
    }

    pub fn closing(&self, actions: &Actions) -> bool {
        if let Some(item) = actions.find_widget_action(self.widget_uid()) {
            if let FoldButtonAction::Closing = item.cast() {
                return true
            }
        }
        false
    }
}
```

### Live Design for FoldButtonWithText

```rust
live_design! {
    FoldButtonWithText = {{FoldButtonWithText}} {
        width: Fit, height: 24
        align: { x: 0.0, y: 0.5 }

        triangle_size: 10.0
        open_text: "Show More"
        close_text: "Show Less"

        draw_bg: {
            instance active: 0.0
            instance hover: 0.0

            fn pixel(self) -> vec4 {
                let sdf = Sdf2d::viewport(self.pos * self.rect_size);
                let sz = self.triangle_size;
                let c = vec2(sz * 0.5, self.rect_size.y * 0.5);

                // Triangle that rotates based on active state
                let rotate = self.active * 1.5708; // 90 degrees

                sdf.rotate(rotate, c.x, c.y);
                sdf.move_to(c.x - sz * 0.3, c.y - sz * 0.4);
                sdf.line_to(c.x + sz * 0.4, c.y);
                sdf.line_to(c.x - sz * 0.3, c.y + sz * 0.4);
                sdf.close_path();

                let color = mix(#666666, #333333, self.hover);
                sdf.fill(color);

                return sdf.result;
            }
        }

        draw_text: {
            text_style: <THEME_FONT_REGULAR> { font_size: 11.0 }
            color: #666666
        }

        animator: {
            hover = {
                default: off
                off = {
                    from: { all: Forward { duration: 0.15 } }
                    apply: { draw_bg: { hover: 0.0 } }
                }
                on = {
                    from: { all: Forward { duration: 0.1 } }
                    apply: { draw_bg: { hover: 1.0 } }
                }
            }
            active = {
                default: off
                off = {
                    from: { all: Forward { duration: 0.2 } }
                    apply: { draw_bg: { active: 0.0 }, active: 0.0 }
                }
                on = {
                    from: { all: Forward { duration: 0.2 } }
                    apply: { draw_bg: { active: 1.0 }, active: 1.0 }
                }
            }
        }
    }
}
```

---

## Common Patterns

### Accordion/Collapsible Section

```rust
live_design! {
    CollapsibleSection = <FoldHeader> {
        header: <View> {
            width: Fill, height: 44
            flow: Right
            align: { y: 0.5 }
            padding: { left: 12, right: 12 }

            show_bg: true
            draw_bg: { color: #f5f5f5 }

            fold_button = <FoldButton> { margin: { right: 8 } }
            section_title = <Label> {
                text: "Section"
                draw_text: {
                    text_style: <THEME_FONT_BOLD> { font_size: 14.0 }
                    color: #333333
                }
            }
        }

        body: <View> {
            width: Fill, height: Fit
            flow: Down
            padding: 12

            // Section content here
        }
    }
}
```

### Color Picker with Fold Header

```rust
live_design! {
    FoldableColorPicker = <FoldHeader> {
        header: <View> {
            width: Fill, height: 40
            flow: Right
            align: { y: 0.5 }
            spacing: 8
            padding: { left: 8, right: 8 }

            fold_button = <FoldButton> {}

            // Color preview swatch
            color_swatch = <View> {
                width: 24, height: 24
                show_bg: true
                draw_bg: { color: #ff0000 }
            }

            // Hex input
            hex_label = <Label> { text: "#" }
            hex_input = <TextInput> {
                width: 70, height: 28
                text: "FF0000"
            }
        }

        body: <View> {
            width: Fill, height: Fit
            flow: Down
            padding: 8
            spacing: 8

            // Full color picker UI
            sv_picker = <View> { /* SV picker */ }
            hue_slider = <View> { /* Hue slider */ }
            presets = <View> { /* Preset colors */ }
        }
    }
}
```

### Programmatic Control

```rust
impl MyWidget {
    fn toggle_fold_header(&mut self, cx: &mut Cx) {
        let fold_button = self.view.fold_button(ids!(my_fold_header.fold_button));

        // Programmatically toggle
        if let Some(mut inner) = fold_button.borrow_mut() {
            if inner.animator_in_state(cx, ids!(active.on)) {
                inner.animator_play(cx, ids!(active.off));
            } else {
                inner.animator_play(cx, ids!(active.on));
            }
        }
    }
}
```

---

## Best Practices

1. **Always include FoldButton**: The header must contain a `FoldButton` for fold behavior

2. **Use `height: Fit` for body**: Allows content to determine height

3. **Use standard FoldButtonAction**: When creating custom fold buttons, use `makepad_widgets::fold_button::FoldButtonAction` for compatibility

4. **No prefix for nested access**: Access widgets directly without "header" or "body" prefixes

5. **Cache expensive computations**: Pre-compute data shown in headers, don't compute during `draw_walk()`

---

## Related Widgets

- `FoldButton` - Triangular fold indicator
- `FoldHeader` - Container with collapsible body
- `PortalList` - Virtualized list (can contain FoldHeaders)
- `View` - Basic container widget
