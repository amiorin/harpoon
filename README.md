# Harpoon Window Manager
## Problem
You have many windows in your space but only some of them are relevant to your work. Using `cmd+back_tick` and `cmd-tab` makes you lose focus and time.

## Solution
Use bookmarks to quickly switch between the relevant windows. In my case I use `F1-5` to navigate to the relevant windows and `S-F1-5` to add the relevant windows. You can repeat `S-F1-5` to cycle through all visible windows of a space. Every space has its own bookmars. You can use multiple spaces for multiple activities.
 
## Installation
1. Use https://github.com/keycastr/keycastr to debug the configuration
1. Install Karabiner to disable `cmd+back_tick` and `cmd-tab`. This will allow to unlearn the anti-pattern.
1. Get used to `mash` (`cmd+opt+ctrl`) and `smash` (`mash` + `shift`). 
1. Map `f1-5` to `mash 1-5` with Karabiner.
1. Map `shift-f1-5` to `smash 1-5` with Karabiner.
1. Map `mash 1-5` to `nav_window` (navigate to a window with a bookmark) with Phoenix.
1. Map `smash 1-5` to `add_window` (add bookmark to a window) with Phoenix.
1. Read https://github.com/jasonm23/Phoenix-config to understand how to convert this `README.md` in a Phoneix config file.

## Alternatives
Buy 3 monitors and use a tiling window manager like https://github.com/koekeishiya/yabai. This was not an option for me.

## Credits
Harpoon Window Manager is built on top of amazing other projects like
1. https://github.com/ThePrimeagen/harpoon (just the main idea)
2. https://github.com/kasper/phoenix
3. https://github.com/jasonm23/Phoenix-config
4. https://karabiner-elements.pqrs.org/

# Code

## Window Grid

Constants

```js @code
MARGIN_X = 0
MARGIN_Y = 0
```

## Helpers

```js @code
Phoenix.notify("Phoenix config loading")

Phoenix.set({
  daemon: false,
  openAtLogin: true
})
```

Logging

```js @code
let log = function (o, label = "XXX") {
  Phoenix.log(`${label} ${JSON.stringify(o)}`)
}
```

Shortcuts for `focused`

```js @code
focused = () => Window.focused()

Window.prototype.screenFrame = function(screen) {
  return (screen != null ? screen.flippedVisibleFrame() : void 0) || this.screen().flippedVisibleFrame()
}

Window.prototype.fullGridFrame = function() {
  return this.calculateGrid({y: 0, x: 0, width: 1, height: 1})
}
```

Calculate the grid based on the parameters, `x`, `y`, `width`, `height`, (returning an object `rectangle`)

```js @code
Window.prototype.calculateGrid = function({x, y, width, height}) {
  return {
    y: Math.round(y * this.screenFrame().height) + MARGIN_Y + this.screenFrame().y,
    x: Math.round(x * this.screenFrame().width) + MARGIN_X + this.screenFrame().x,
    width: Math.round(width * this.screenFrame().width) - 2.0 * MARGIN_X,
    height: Math.round(height * this.screenFrame().height) - 2.0 * MARGIN_Y
  }
}
```

## Window moving and sizing

Temporary storage for frames

```js @code
lastFrames = {}
```

Window to grid

```js @code
Window.prototype.toGrid = function({x, y, width, height}) {
  let rect = this.calculateGrid({x, y, width, height})
  return this.setFrame(rect)
}
```

Toggle a window to full screen or revert to it's former frame size.

```js @code
Window.prototype.toFullScreen = function(toggle = true) {
  if (!_.isEqual(this.frame(), this.fullGridFrame())) {
    this.rememberFrame()
    return this.toGrid({y: 0, x: 0, width: 1, height: 1})
  } else if (toggle && lastFrames[this.uid()]) {
    this.setFrame(lastFrames[this.uid()])
    return this.forgetFrame()
  }
}
```

Remember and forget frames

```js @code
Window.prototype.uid = function() {
  return `${this.app().name()}::${this.title()}`
}

Window.prototype.rememberFrame = function() {
  return lastFrames[this.uid()] = this.frame()
}

Window.prototype.forgetFrame = function() {
  return delete lastFrames[this.uid()]
}
```

## Harpoon

This is the logic for the Harpoon Window Manager

```js @code
harpoon = Storage.get('harpoon')
modal = {n: null, ts: 0}
timeout = 2000

const showPopup = str => {
  let frame = Screen.main().frame()
  let modal = Modal.build({
    duration: 1.0,
    text: str
  })
  modal.origin = {
    x: (frame.width / 2) - modal.frame().width / 2,
    y: frame.height - 100
  }
  modal.show()
}

const init_harpoon = (space_id, n) => {
  if (!_.isObject(harpoon)) { harpoon = {} }
  if (!_.isObject(harpoon[space_id])) { harpoon[space_id] = {} }
  if (!_.isObject(harpoon[space_id][n])) {
    harpoon[space_id][n] = {index: -1, win_id: null}
  }
}

const add_window = n => {
  if (modal.n != n || Date.now() - modal.ts > timeout) {
    showPopup(n)
  }
  const space_id = Space.active().hash()
  init_harpoon(space_id, n)
  const wins = Space.active().windows({visible: true})
  const focused_id = focused().hash()
  if (wins.length == 0) { return }
  if (modal.n != n && Date.now() - modal.ts < timeout) {
    // assign to the focused window
    let index
    _.each(wins, (win, key) => {
      if (focused_id == win.hash()) {
        index = key
      }
    })
    harpoon[space_id][n].index = index
    harpoon[space_id][n].win_id = focused_id
  } else {
    // cycle and then assign
    index = harpoon[space_id][n].index + 1
    index = index % wins.length
    if (focused_id == wins[index].hash()) { 
      index += 1
      index = index % wins.length
    }
    wins[index].focus()
    harpoon[space_id][n].index = index
    harpoon[space_id][n].win_id = focused().hash()
  }
  Storage.set('harpoon', harpoon)
  modal.n = n
  modal.ts = Date.now()
}

const nav_window = n => {
  const space_id = Space.active().hash()
  init_harpoon(space_id, n)
  let win_id = harpoon[space_id][n].win_id
  if (_.isNull(win_id)) {
    showPopup("404")
    return
  }
  let found = false;
  _.each(Space.active().windows({visible: true}), win => {
    if (win.hash() == win_id) {
      win.focus()
      found = true;
    }
  })
  if (!found) { showPopup("404") }
}
```

### App Name Modal

```js @code
let showAppName = () => {
  let name = focused().app().name()
  let frame = focused().screenFrame()
  let modal = Modal.build({
    duration: 2,
    text: `App: ${name}`
  })
  modal.origin = {
    x: (frame.width / 2) - modal.frame().width / 2,
    y: frame.height - 100
  }
  modal.show()
}
```

(It's  pretty cool, but it's clearly a bezel ;)

### Binding alias

Alias `Phoenix.bind` as `bind_key`, to make the binding table extra
readable.

```js @code
keys = []
```

The `bind_key` method includes the unused `description` parameter,
This is to allow future functionality i.e. help mechanisms, describe bindings etc.

```js @code
const bind_key = (key, description, modifier, fn) => keys.push(Key.on(key, modifier, fn))
```

## Bindings

Mash is <kbd>Cmd</kbd> + <kbd>Alt/Opt</kbd> + <kbd>Ctrl</kbd> pressed together.

```js @code
const mash = 'cmd-alt-ctrl'.split('-')
```

Smash is Mash + <kbd>shift</kbd>

```js @code
const smash = 'cmd-alt-ctrl-shift'.split('-')
```

Toggle maximize for the current window

```js @code
bind_key('M', 'Maximize Window', mash, () => focused().toFullScreen())
```

Harpoon shortcuts

```js @code
bind_key('1', 'Nav F1', mash, () => nav_window("F1"))
bind_key('1', 'Add F1', smash, () => add_window("F1"))
bind_key('2', 'Nav F2', mash, () => nav_window("F2"))
bind_key('2', 'Add F2', smash, () => add_window("F2"))
bind_key('3', 'Nav F3', mash, () => nav_window("F3"))
bind_key('3', 'Add F3', smash, () => add_window("F3"))
bind_key('4', 'Nav F4', mash, () => nav_window("F4"))
bind_key('4', 'Add F4', smash, () => add_window("F4"))
bind_key('5', 'Nav F5', mash, () => nav_window("F5"))
bind_key('5', 'Add F5', smash, () => add_window("F5"))
```

All done...

```js @code
Phoenix.notify("All ok.")
```
