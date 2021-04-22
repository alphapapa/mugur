# Mugur

[![MELPA](https://melpa.org/packages/mugur-badge.svg)](https://melpa.org/#/mugur)

An Emacs configurator for [QMK](https://github.com/qmk/qmk_firmware) compatible
keyboards.

# Table of Contents

- [Mugur](#mugur)
- [How it works](#how-it-works)
- [Supported QMK keycodes and features](#supported-qmk-keycodes-and-features)
  * [Basic Keycodes](#basic-keycodes)
  * [Mod-tap](#mod-tap)
  * [Modifiers](#modifiers)
  * [One Shot Keys](#one-shot-keys)
  * [Layer Toggle](#layer-toggle)
  * [Macros](#macros)
  * [Tap Dance](#tap-dance)
  * [Leader Key](#leader-key)
  * [Emacs keybound functions](#emacs-keybound-functions)
  * [User Defined Keys](#user-defined-keys)
- [Configuration](#configuration)
  * [Paths and names](#paths-and-names)
  * [Rules (rules.mk)](#rules--rulesmk-)
  * [Configs (config.h)](#configs--configh-)
  * [Others](#others)
- [Supported Keyboards](#supported-keyboards)
- [Other points](#other-points)
- [An Extended Example for Ergodox](#an-extended-example-for-ergodox)

(Table of contents generated with [markdown-toc](http://ecotrust-canada.github.io/markdown-toc/))

# How it works

Given a list of all the layers and keys, `mugur-mugur` generates the equivalent
C code for the qmk_firmware, ready to be built and flashed on the keyboard.

```emacs-lisp
(mugur-mugur
 '(("base"    a b c (LT numbers d))
   ("numbers" 1 2 3 "one two three")))
```

This is a two-layers layout for a keyboard with four keys. The "base" layer has
the letters 'a', 'b' and 'c' and a layer-toggle key that activates the "numbers"
layer when held and sends the letter 'd' when tapped. On the "numbers" layer
there are the 1, 2 and 3 digits, as expected, and then a macro key that sends
"one two three" when pressed.

There are other possibilities, qmk features and configurations to try, but that
is the basics of it. See [an extended example for
ergodox](#an-extended-example-for-ergodox) for a more rich mugur layout.

# Supported QMK keycodes and features

mugur-key is just a term I've invented for any of the symbols, characters,
strings or lists that can appear in a mugur-keymap. In the above example, both
'a' and "one two three" are valid mugur-keys.

## Basic Keycodes
Mugur supports all the [basic qmk
keycodes](https://docs.qmk.fm/#/keycodes_basic).  Their mugur-key equivalent is
either a symbol, a digit, a character or a string.

The mugur-keys are case sensitive, c and C being different mugur-keys, one for the
letter 'c', the other for the Ctrl modifier. Below is the list of all modifiers
plus some additional examples.

| mugur-key | qmk-keycode | comment                             |
|:----------|:------------|-------------------------------------|
| C         | KC_LCTL     | the C (or Ctrl) modifier            |
| M         | KC_LALT     | the M (or Meta) modifier            |
| G         | KC_LGUI     | the G (or Super) modifier           |
| S         | KC_LSFT     | the S (or Shift) modifier           |
| c         | KC_C        | the letter c, given as a symbol     |
| "x"       | KC_X        | the letter x, given as a string     |
| ?\+       | KC_PLUS     | the plus sign, given as a character |
| f12       | KC_F12      | the function key F12                |
| 9         | KC_9        | the digit 9                         |
| xx        | nil         | Invalid key, results in error       |

For every mugur-key there is either a direct correspondence with a qmk-keycode
or with a qmk functionality (like a macro, for example). For a list of all these
basic codes that one can use in the layout, consult the `mugur-symbol` function
as defined in mugur.el. For the special keys, like macros, one-shot and the
rest, see below.

## Mod-tap
When pressed, the modifier is active, when tapped the key is sent.

| mugur-key | example comment                              |
|:----------|:---------------------------------------------|
| (C x)     | Ctrl modifier when held, send x when tappend |
| (S x)     | Hold Shift when held, send x when tapped     |
| (C M x)   | C+M modifier when held, send x when tapped   |
| (C M)     | Invalid, two modifiers given                 |
| (C a b)   | Invalid, more than one key given             |
|           |                                              |

## Modifiers
Allows key combinations like the familiar emacs's M-x, that is, hold down the
Meta modifier and press x.

| mugur-key | example comment                          |
|:----------|:-----------------------------------------|
| M-x       | Send x with Meta held down               |
| C-M-x     | Send x with both Ctrl and Meta held down |
| C-M-yes   | Invalid keycode, results in error        |

## One Shot Keys
One shot keys are keys that remain active until the next key is pressed, and
then are released ([qmk documentation](https://docs.qmk.fm/#/one_shot_keys))

| mugur-key     | example comment                                               |
|:--------------|:--------------------------------------------------------------|
| (OSM S)       | One Shot Mod - the next key will be pressed with Shift active |
| (OSL mylayer) | One Shot Layer - the next key will be from this layer         |

## Layer Toggle
Switching and toggling layers. 

| mugur-key      | example comment                                                                             |
|:---------------|:--------------------------------------------------------------------------------------------|
| (DF numbers)   | switch to 'numbers' layer                                                                   |
| (MO numbers)   | momentarily activates the 'numbers' layer (as long as the key is kept pressed)              |
| (LM numbers C) | momentarily activates the 'numbers' layer but with the C modifier active                    |
| (LT numbers x) | momentarily activates the 'numbers' layer when held and sends x when tapped                 |
| (OSL numbers)  | momentarily activates the 'numbers' layer until the next key is pressed                     |
| (TG numbers)   | toggle the numbers layer, activating it if its inactive and vice-versa                      |
| (TO numbers)   | activates the numbers layer and deactivates all the other layers, besides the default layer |
| (TT numbers)   | layer tap-toggle                                                                            |

## Macros
Send any key combination that does not qualify as anything else.

| mugur-key               | example comment                                                                   |
|:------------------------|:----------------------------------------------------------------------------------|
| "this is mugur!"        | send the specified string when key is tapped                                      |
| (C-x d)                 | send x with C modifier pressed and then send d. (the Emacs dired command)         |
| (C-M-u "my pass" enter) | send u with both C and M modifier pressed, send "my pass" and press enter         |
| ">"                     | this is *not* a macro since it has a basic qmk-keycode to which it is transformed |
| "λ"                     | but this one is a macro, and this key will insert the beloved lambda              |
| (C-x dd)                | invalid, since dd is not a valid mugur-key. Results in error                      |

## Tap Dance
Hit a key once, send a key. Hit it twice in quick succession, send a different
one. The tap dance functionality is only valid for two basic mugur-keycodes
(letter, digits, punctuation, commands, etc), and not for layers nor
macros. `mugur-tapdance-enable` must be set to "yes" for this to work, otherwise
look out for `error: implicit declaration of function ‘ACTION_TAP_DANCE_DOUBLE’`
when building.

| mugur-key        | example comment                                                                   |
|:-----------------|:----------------------------------------------------------------------------------|
| (DANCE a x)      | Send `a` when hitting the key once, send `x` when hit twice, in quick succession. |
| (DANCE ?\: ?\;)  | Send `;` when hitting the key once, send `;` when hit twice, in quick succession  |
| (DANCE a "this") | Invalid, mugur only supports basic mugur-keycodes, not macros, nor layers, etc.   |
| (DANCE a b c)    | Invalid, can't do more than two things.                                           |

## Leader Key
Tap the Leader Key and then up to five keys in quick succession, before the
`LEADER_TIMEOUT` expires, and send whatever macro you like. Use the
`mugur-leader-keys` to setup what keys do what

```emacs-lisp
(setf mugur-leader-keys
      '(((s d f) "s, d and then f")
        ((z)     (M-x "kill"))))
```

In the above example, tapping the Leader Key, followed by `s`, `d` and then `f`
will send the given string. The mugur-key after the first list, must be a valid
[macro](#macros).

## Emacs keybound functions
For Emacs functions that have a keybinding, the function name can be directly specified as a mugur-key.

| mugur-keycode | example comment                                                                     |
|:--------------|:------------------------------------------------------------------------------------|
| query-replace | the usual keybinding for this is M-%, so this is equivalent with the M-% mugur-key  |
| insert-char   | Becomes the mugur-key '(C-x 8 RET), which, in turn, is a macro that send those keys |
|               |                                                                                     |

## User Defined Keys
For long or often-used mugur-key sequences, or simply as a mnemonic, the
`mugur-user-defined-keys` contains a list of items, where the car of the item is
the mnemonic and the cadr is any valid mugur-key.

```emacs-list
(setf mugur-user-define-keys
      '((uname        "my_badass_username")
        (weird_key    (C-c a "right?" ENT))))
```

# Configuration

## Paths and names

**mugur-qmk-path** *nil*

    Path to the qmk firmware source code (root folder).

**mugur-keyboard-name** *nil*

    The name of your qmk keyboard."

**mugur-layout-name** *nil*

    The keymap name used in the keymaps matrix.
    Check the 'uint16_t keymaps' matrix in the default keymap.c of
    your keyboard.  Some have just "LAYOUT", others
    "LAYOUT_ergodox", etc. Adapt accordingly.

**mugur-keymap-name** *nil*

    The name of qmk keymap generated by mugur.

## Rules (rules.mk)
          
Generated based on the mugur-keys.

| mugur-key                   | rule automatically included in rules.mk |
|:----------------------------|:----------------------------------------|
| lead                        | LEADER_ENABLE = yes                     |
| (DANCE mugur-key mugur-key) | TAP_DANCE_ENABLE = yes                  |
| rgb_tog                     | RGBLIGHT_ENABLE = yes                   |
|                             |                                         |


If 'lead (leader key) is available on the mugur-keymap, 
          
## Configs (config.h)
    
**mugur-tapping-term** 180

    Tapping term, in ms.
    This is the maximum time allowed between taps of your Tap Dance
    mugur-key.

**mugur-leader-timeout** 300
    
    Timeout for the Leader Key, in ms.
    This is the time to wait to complete the Leader Key sequence
    after you press the Leader Key key.

**mugur-leader-per-key-timing** nil

    Enable or disable LEADER_PER_KEY_TIMING option.
    If enabled, the Leader Key timeout is reset after each key is
    tapped.

## Others

**mugur-user-defined-keys** nil

    User defined keys for long or often used combinations.
    A list of lists, where the car of each antry is a symbol (a name
    for the new key) and the cadr is any valid mugur-key.

**mugur-ignore-key** *'-x-*

    The symbol used in the mugur-keymap for an ignored key.
    This is transformed into the qmk-keycode KC_NO.
    
**mugur-transparent-key** *'---*

    The symbol used in the mugur keymap for a transparent key.
    This is transformed into the qmk-keycode KC_TRANSPARENT.

# Supported Keyboards

Any keyboard that uses the QMK firmware. Users have reported success using mugur
for the following keyboards,

| QMK Keyboard | Comments |
|:-------------|:---------|
| Ergodox (EZ) |          |

Send me an email if you've used mugur for other keyboards successfully. If
you have problems with a particular keyboard, open an issue with the keyboard
name and the particular problem you're facing.

# Other points

* Not all qmk features are supported, nor there is a plan to support them all.
* The layout structure that mugur accepts is the layout that qmk keymap.c is
  using for your particular keyboard (check the qmk_firmware sourcecode). There
  is a one-to-one correspondence to what is defined in a mugur-layout and what
  you see in the qmk-layout. In short, no vertical or horizontal reordering of
  the keys is supported nor planned to.
* There is no support to define which LEDs to turn on/off for which mugur-layer,
  nor there is a plan to support such a feature.
* The mugur package generates the qmk-layout, but does not build nor flashes it
  on your keyboard. The layout name and location is based on the values of the
  custom variables given by the user (see the [configuration](#Configuration)
  section). Consult the qmk documentation and your keyboard README for how to
  build and flash your keyboard.

# An Extended Example for Ergodox

```emacs-lisp
(let ((mugur-qmk-path        "/home/mihai/projects/qmk_firmware")
      (mugur-keyboard-name   "ergodox_ez")
      (mugur-layout-name     "LAYOUT_ergodox")
      (mugur-keymap-name     "mugur_test")
      (mugur-tapping-term    175)
      (mugur-user-defined-keys '((email  "mihai@fastmail.fm")
                                 (replay (C-x e)))))
                                 
  (mugur-mugur
   '(("base" 
      C-f1    vol-down  vol-up   -x-   -x-  -x- reset
       -x-     (C-x k)     w      e     r    t  replay
       -x-        a      (G s)  (M d) (C f)  g 
     (OSM S)      z        x      c     v    b  email
       -x-       -x-      -x-    -x-   tab
     
                                           tab   -x-
                                                 -x-
                                   bspace space  -x-
       
       -x- -x-  -x-     -x-      -x-  -x-    -x-
       -x-  y    u   (LT NUM i)   o   -x-    -x-
            h  (C j)   (M k)    (G l)  p     -x-
       -x-  n    m     comma     dot   q   (OSM S)
                -x-      up     down  left  right
     
       -x-               -x-
       (C-x b)
       (LT media escape) (LT movement ent) (LT symbols pscreen))

     ("num"
      --- --- --- --- --- --- ---
      --- ---  1   2   3  --- --- 
      ---  0   4   5   6  --- 
      ---  0   7   8   9  --- --- 
      --- --- --- --- --- 
                          --- --- 
                              ---
                      --- --- ---

      --- --- --- --- --- --- ---
      --- --- --- --- --- --- ---
          --- --- --- --- --- ---
      --- --- --- --- --- --- ---
              --- --- --- --- ---
      --- ---
      ---
      --- --- ---)

     ("movement"
      ---  ---       ---      ---     ---    --- --- 
      ---  ---  (C-u C-space)  up   S-insert M-< --- 
      ---  C-a      left      down   right   C-e 
      ---  undo      ---      ---     ---    M-> ---
      ---  ---       ---      ---     --- 
      ---  --- 
      ---
      delete C-tab ---
    
      --- ---      ---                ---                 ---       --- ---
      --- ---      ---          sp-backward-up-sexp       ---       --- ---
          --- sp-backward-sexp        ---           sp-forward-sexp --- ---
      --- ---      ---                ---                 ---       --- ---
                   ---                ---                 ---       --- ---
      --- ---
      ---
      --- --- ---)

     ("symbols"
      ---  ---  ?\[  ?\]  ?\{  ?\}  ---
      ---  ?\~  ---  ?\'  ?\"  ?\`  ---
      ---  ?\;  ?\:  ?\-  ?\(  ?\) 
      ---  ?\/  ?\\  ?\=  ?\+  ?\_  ---
      ---  ---  ?\|  ?\<  ?\> 
      ---  ---
      ---
      ?\! ?\? ---
    
      --- --- --- --- --- --- ---
      --- --- --- --- --- --- ---
          --- --- --- --- --- ---
      --- --- --- --- --- --- ---
      --- --- --- --- ---
      --- ---
      ---
      --- --- ---)

     ("media"
      ---       ---        ---     ---        ---     --- --- 
      rgb_sad  rgb_sai     ---    ms_up    ms_wh_up   --- ---
      rgb_vad  rgb_vai  ms_left  ms_down   ms_right   --- 
      rgb_hud  rgb_hui     ---     ---    ms_wh_down  --- ---
      rgb_tog    ---       ---     ---        --- 
      --- ---
      --- 
      ms_btn2  ms_btn1  ms_btn3
    
      --- --- --- --- --- --- ---
      --- --- --- --- --- --- ---
          --- --- --- --- --- ---
      --- --- --- --- --- --- ---
              --- --- --- --- ---
      --- ---
      ---
      --- --- ---))))
```
