---
title: "Project: GIS"
date: 2024-01-04
---

Project date: ~2021

There is something very satisfying watching a bot to do tedious tasks for you while you sit back and relax. You feel like you're making progress without lifting a finger.

![Automation](https://imgs.xkcd.com/comics/automation.png)

# Changing the Active Window, on Windows

I enjoy automating things. Something that's common to nearly every task is getting and changing the active window.

![The alt-tab experience](https://www.windowslatest.com/wp-content/uploads/2020/07/Alt-Tab-with-browser-tabs-1024x572.jpg)

Python is usually my language of choice for these things because 1) writing Python code is just a delight and 2) Python being as popular as it is, has lots of support for automation. 

Here are three packages that can all accomplish the task for us:
- **[PyAutoGUI](https://github.com/asweigart/pyautogui)** - the package featured on "[automate the boring stuff with python](https://automatetheboringstuff.com/2e/chapter20/)"
- **[pywinauto](https://github.com/pywinauto/pywinauto)** - the premier automation package for windows UI elements
- **[ahk](https://github.com/spyoungtech/ahk)** - a Python wrapper for [AutoHotkey](https://www.autohotkey.com/), an open-source automation scripting language

```python
# PyAutoGUI
win = pyautogui.getWindowsWithTitle('Untitled - Notepad')[0]
win.activate()

# pywinauto
# handle = pywinauto.findwindows.find_window(title='Untitled - Notepad')
# app = pywinauto.application.Application().connect(title='Untitled - Notepad')
app = pywinauto.Application().connect(title="Untitled - Notepad")
app.UntitledNotepad.set_focus()

# ahk
win = ahk.find_window(title='Untitled - Notepad') # Find the opened window
win.activate()
```

These packages are great but they can be overkill depending on your need. They come loaded with a bunch of features that we may simply never need. 

To write our own window-activator, we'll use the pywin32 package which provides acess to Windows APIs. We could use ctypes as well but I think this method is more accessible. 

All we need are two functions: [FindWindow](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-findwindowa) and [SetForegroundWindow](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setforegroundwindow).

```python
handle = win32gui.FindWindow(None, "Untitled - Notepad")
win32gui.SetForegroundWindow(handle)
```

Easy. Now we can activate any window we desire. But what if I want to change focus to a different window while I'm actively using a *different* window?

Going back to the documentation for [SetForegroundWindow](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setforegroundwindow), it says:

>An application cannot force a window to the foreground while the user is working with another window. Instead, Windows flashes the taskbar button of the window to notify the user.

The only exception is that when `alt` key is pressed, you are allowed to change the window focus. Using this information, people have created a workaround.

```python
# Modified from https://stackoverflow.com/a/61180328
import win32gui, win32com.client
# Focus the desktop
shell = win32com.client.Dispatch("WScript.Shell")
# Press the Alt key
shell.SendKeys('%')
# Activate the target window
win32gui.SetForegroundWindow(handle)
```

Note: Since this project, I have found a [different approach](https://stackoverflow.com/questions/688337/how-do-i-force-my-app-to-come-to-the-front-and-take-focus/20691831#20691831) online:

>1. Attach to the thread of the window that currently has focus.
>2. Bring your window into focus.
>3. Detach from the thread.

I haven't tried it yet but this method may have more merit.


# Wrapping Up

Now that we know how to find and activate a window, we can do cool things with it.

We can create a Window class:

```Python
class Window:
    def __init__(self, handle) -> None:
        self.handle = handle
	
	def activate(self) -> None:
		shell = win32com.client.Dispatch("WScript.Shell")
		shell.SendKeys("%")
		win32gui.SetForegroundWindow(self.handle)
```

And a helper function:

```Python
def find_window(class_name, window_title) -> Window:
    handle = win32gui.FindWindow(class_name, window_title)
    return Window(handle)
```

Or extend the Window class:

```Python
class Window:
	@contextmanager
	def activate_momentarily(self, *args, **kwargs) -> None:
		# Save the current window.
		prev_window = get_foreground_window()		
		try:
			# Activate the target window.
			self.activate(*args, **kwargs)
			yield
		finally:
			# Return control to the previous window.
			prev_window.activate(*args, **kwargs)
```

So we can do this:

```python
telegram = find_window(None, "Telegram")
with telegram.activate_momentarily():
	telegram.write("Hello World!")
```

[View on GitHub](https://github.com/hansjunodev/window-utils)




