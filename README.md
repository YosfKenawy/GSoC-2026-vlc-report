# GSoC 2026 with VideoLAN

**Student Name** : Yousef Kenawy

**Email** : yosefahmed77310@gmail.com

**Organization** : VideoLAN

**Project** :  Qt Integration Tests

**Mentor** : Pierre Lamot

## Project Overview

This project aims to add automated tests to check the behaviour of UI components using the VLC media player. These tests are based on a UI testing framework that simulates user interaction with a mouse and keyboard. The framework creates a separate pseudo sandbox, deletes all VLC data, then generates new data with a media library generation module so the tests dont interfer with VLC data. This project also introduces a new accessibility IDs `AccessibleCompat.id` so that UI elements can be found by automated tests 

## Work timeline

### Technologies and Tools Used

- **Python 3** 

- **dogtail 2.0** (AT-SPI / accessibility) 

- **VLC HTTP interface** (`--extraintf=http`)

- **Qt 6.11.1**

- **Accerciser** for AT-SPI tree inspection during development

### Compiling VLC

I used Ubuntu to work on VLC, i usually didn't use Linux a lot before GSoC, so i think it was a great opportunity to get into linux.

I was able to compile VLC with no problems, later when i rebuilt it i would get errors for deleted files or a wrong path i used for the build command which was a problem when i was compiling VLC with Media Library and it worked by using `--enable-medialib-gen`, with time i knew how to manage the builds better and how to deal with each of these problems.

My first challenge was getting VLC to compile with Qt 6.11.1, which required rebuilding Qt from source and then rebuilding VLC against it.

### Preparation

In the Community Bonding Period, I needed to understand how the framework worked. The framework was built by Wassim Lalaoui, whose merge requests ([!8450](https://code.videolan.org/videolan/vlc/-/merge_requests/8450)
and [!8460](https://code.videolan.org/videolan/vlc/-/merge_requests/8460)) provided the initial test infrastructure that includes a `VLCTests` base class that handles test setup, a `LinuxDriverUtils` class that wraps dogtail for AT-SPI interaction, and sandboxed temp directories.

I studied how the AT-SPI accessibility works on Linux, it exposes VLC's entire UI tree, and each QML element that has an `AccessibleCompat.id` set that can be found by using dogtail. This is much more reliable than searching by name or position, because the ID is stable across layout changes.

### Adding Accessibility IDs

Merge Request :  [accessibility-ids](https://code.videolan.org/videolan/vlc/-/merge_requests/9702)

I started my work with adding the Accessibility IDs, i checked every QML component in VLC's interface and added  104 `AccessibleCompat.id` to 70 QML files:

- [Playlist and network components](https://code.videolan.org/videolan/vlc/-/merge_requests/9702/diffs?commit_id=a20a8bc26cb6ad1c96b8002be746049ce63e3112)
- [Medialibrary components](https://code.videolan.org/videolan/vlc/-/merge_requests/9702/diffs?commit_id=22fe32e56c69acb67db6e737911fd8e8da1c85d8)
- [Navigation and sidebar components](https://code.videolan.org/videolan/vlc/-/merge_requests/9702/diffs?commit_id=d9077b4f51ac0c70bbc4d360ecab764791ef0b57)
- [Player controls](https://code.videolan.org/videolan/vlc/-/merge_requests/9702/diffs?commit_id=215d1feabc7d0106643463d5580470141ffc6e1d)

The naming convention was important to prevent any redundant overrides and give more uniqueness to each id

### Writing the Tests

Merge Request : [ui-tests](https://code.videolan.org/videolan/vlc/-/merge_requests/9836)

I created six test files, each focused on a specific area of VLC's functionality.

**test_player_controls.py**: covers all major control buttons and detects each button accessibility and behaviour (uses a customized controlbar i will talk about it later)

**test_keyboard_navigation.py**: These tests verify that VLC's keyboard navigation works correctly with the focus movement, which is important for accessibility

**test_keyboard_shortcuts.py**: tests core shortcut behaviour (`s` shortcut for stopping playback) across five different  states, player view with embedded video, player view with music, player view with not embedded video (using `--no-embedded-video`), medialibrary view, and minimal mode (using `--qt-minimal-view`). Also tests Ui shortcuts, but it is disabled because VLC's Global shortcuts don't fire during tests.

**test_playqueue.py**:  tests the playqueue panel behaviour, verifying current and enqueued tracks are shown, and testing loop/shuffle toggles within the panel.

**test_basics.py**: Verifies VLC starts without crashing, the main window is visible, and tabs and subtabs navigation.

**test_favorites.py** : disabled because of this Qt bug: https://bugreports.qt.io/browse/QTBUG-110624 that prevent context menus to be accessible with AT-SPI.

### The Toolbar Customization

I had a problem with testing all the buttons while most of them don't exist in the default toolbar, so I wrote a `customize_toolbar()` function in linux/utils.py that takes a list of ControlType enum values for each button, converts them to their integer representations, and writes them under the ControlbarProfileModel group before VLC launches.

The format puts all requested buttons into the center zone for all three modes (Videoplayer, Audioplayer, Miniplayer), so every test sees the same full set of controls regardless of what media is playing.

When a test method declares a controlbar attribute, `_apply_controlbar()` reads it during `setUp()` and calls `customize_toolbar()` with that list. This means individual tests can request exactly the buttons they need.

### The HTTP Status

The tests needed a way to check the state of each action done: is the video actually paused or the player view in fullscreen? And because the accessible value isn't exposed, so by using the HTTP server interface in VLC made it easier to get the state of the player by using `--extraintf=http`, which exposes a status.xml endpoint that returns the current playback state as XML

I built an `HttpStatus` class that reads this endpoint with an HTTP basic auth handler, parses the XML, and exposes methods like get_state(), get_volume(), get_loop(), and more to be used for different tests. This catches cases where the UI looks right, but the backend is out of sync.

### Windows Testing 

I built VLC on Windows so i can run the tests there. The process was much easier compared to Linux, i followed this document [BUILD-win32](https://code.videolan.org/videolan/vlc/-/blob/master/doc/BUILD-win32.md) and used an Ubuntu WSL and LLVM prebuilt, and i didn't get a lot of blockers this time. 

VLC doesn't have a Windows runner at the moment, so the CI for Windows would be left disabled.

## What Needs To Be Done

Right now the core framework and 41 active tests are working, several areas need more work:

**Un-disabling tests**: 18 Qt UI shortcut tests are disabled because GlobalShortcuts don't fire during tests. I also had this problem when i ran VLC outside of the test framework, where the shortcuts don't fire also, i found that `ShortcutExt.qml` registers every shortcut with `Qt.ApplicationShortcut` but the problem is that under Xvfb there is no window manager, so the VLC window is never active even if there is focus on the window, the shortcuts don't fire.
Additionally, mute tests are blocked because VLC's HTTP `status.xml` doesn't expose a `<mute>` element, so it needs to be added in the Lua HTTP interface and then add the mute tests.

**Adding more tests**: More tests can be added, especially in keyboard navigation and other UI parts like the media library and menus

**Windows tests validation**: The framework works on Windows, but it would need more work to validate all the tests and utilities to make them work on Windows.

**CI adjustment**: Run the tests using the CI images to make sure it works well and adjust the CI to work well in VLC runners (Linux)

## Video Demo

https://github.com/user-attachments/assets/e90507b7-f868-43ae-ac20-81a375e70bbb

60 tests total: 41 passing, 19 skipped, 0 failures.

## Conclusion

My experience with this project and working with VLC was really beneficial in many aspects.
Technically i learned a lot about git,this helped to manage my commits and merge request to be more efficiently. Also i understood more about testing and how important it is for adding new features and making sure the UI behaviour works well. The most important part was the communication and discussions, and that was really inspiring for me to contribute more.

I'd like to thank my mentor Pierre Lamot for guiding me through this project. Pierre was really responsive to my questions and helped me a lot with many blockers i had. I'd also like to thank the 


VideoLAN organization and their community, i admire their work and how helpful and encouraging the other members are. I am grateful that i had the opportunity to
contribute to VLC.















