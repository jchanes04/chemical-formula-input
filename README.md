# Chemical Formula Input

## TODO

- Better scrolling of overflowing text
- Styling/settings
- Chemical formula substitutions/formatting
- Cursor movement controls
- Accessibility
- Mobile/cross-browser testing
- Copy/cut/paste

## Planned Settings

- Cursor component
- Normal/subscript/superscript text component
- Symbol dictionary (what to convert to what, what to automatically super/subscript, etc.)
- Selection color

## Cursor Selection Notes

1. Place cursor at beginning point (already happens)
2. Place invisible spans one character before and after the cursor
3. Watch for mousemove and use mouse position + getClientRect to determine which characters the cursor is most nearly between
4. When the selection position changes, move spans to the left and right of new selection end position, and repeat until mouseup
