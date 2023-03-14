# sd_extension_template

- Create templates for strangely stuck places in extension development
- Official documentation
  - https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Developing-extensions

# supported version

## Python

### Compatible with Python 3.8+

- Only Windows can install 3.10 easily
- Some environments such as Colab and ROCm still have 3.8 users
- conda users can only go up to 3.9
- It seems difficult to coexist with pip and xformers
- EoL of 3.8 seems to be October 2024...

Difference between 3.8 and 3.9 and 3.10

- __file__ in 3.8 can be absolute path with relative path os.path.abspath(__file__)
- Since match syntax cannot be used before 3.9, use if instead

## Git

- I want to install lfs on my own, but it feels like overkill.

## CUDA

- 11.6 seems fine

## Modules listed in requirements.txt

- It is safe to follow the main version
- The main unit upgrades on its own, but I have no choice but to obey

# file name and location

## repository name

- It is safer to use underscores instead of hyphens.
- Importing directories with hyphens requires importlib.

## Rename folder containing extension

- Although it can be specified as a function of the 1111 main unit, it is quite difficult to handle.
  - For example sd_drebooth_extension does not work.
- As an author, I will do my best to respond.
- As a user, it is safer to use without changing the name.

## install.py

- Called at install time and every boot.
- So heavy processing is strictly prohibited.

The basic is to use launch.py and perform pip install.

```python
import launch

if not launch.is_installed("yaml"):
    launch.run_pip("install yaml", desc='yaml')
```

For complex processing, it is good to put a file that will be a flag when successful.

```python
import pathlib
p = pathlib.Path(__file__).parts[-3:-1]
checked_path = os.path.join(p[0], p[1], 'install.checked')
if not os.path.exists(checked_path):
    # 込み入った処理...
    pathlib.Path(checked_path).write_text('1')
```

- cannot use scripts.basedir() in install.py

## preload.py

- A feature requested by the author of sd_drebooth_extension.
- Basically, it is better not to use it.
  - 1111 main unit does not start when extension is removed after adding command line option.
  - There is no way to know your own location.

## put py files in scripts/

- File name can be anything.
- The types are divided as follows (there may be others).
  - API: script_callbacks.on_app_started(APIfunc)
    - Start with --api option
  - Tab: script_callbacks.on_ui_tabs(on_ui_tabs)
    - UI displayed as tabs on the screen
  - Script: class Script(scripts.Script)
    - UI to select in Script dropdown for txt2img/img2img
  - Others: Items without the above three specifications
- Presumably all files are read once when gradio starts.

- Most used tabs
  - At least keep the tab names short, as it's full of tabs and gets in the way

### imports in scripts/

```python
from scripts import foo, bar, baz
```

### How to extend a single script file

- Just put scripts/foo.py into git repository
- 1111 will do the rest

## put py files outside scripts/

- For example you want to create a py/ directory.
- from py to see the py directory from scripts

```python
from py import foo, bar, baz
```

### Import another file in the same directory in the py directory

```python
from . import foo, bar, baz
from .foo import foofanc
```

### import py file in another directory from py directory

- Easier not to

```python
def example_func():
    import pathlib
    p = pathlib.Path(__file__).parts[-4:-2]
    import importlib
    example3 = importlib.import_module(f"{p[0]}.{p[1]}.py2.example3")
```

### Extending Python written for the command line

- Bring and place files
- Replace hyphens in repository names with underscores if possible
- Rewrite all import statements to specify relative paths
- Call the contents of __main__ from your own UI with import or importlib
- Make it possible to input sys.argv and its parser part by UI or file

## place a file, output a file

- It's better to put it in Extension.

```python
def get_input_dir():
    import pathlib
    p = pathlib.Path(__file__).parts[-4:-2]
    input_dir = os.path.join(p[0], p[1], 'input')
    return input_dir
```

### Place config and log files

- Do not put the file in git (because it will be overwritten and initialized when pulling)
- Better to make a directory

```python
def get_config_path():
    import pathlib
    p = pathlib.Path(__file__).parts[-4:-2]
    config_path = os.path.join(p[0], p[1], 'json', 'config.json')
    return config_path
```

# Development environment

## Debugging with VSCode

- I can't
- Probably it seems necessary to edit as a project loaded with 1111
- Note that venv will be dirty if you install development tools

## Restart UI is not available

- Even if you update the file, it is not reflected
- Do not write processing that causes an error if the same code is executed twice
  - It is better not to create pseudo-Const classes
- You can skip loading the model with --debug-ui, but you can't use it with an extension that requires a model
- Startup time is only reduced by about 5 seconds even with null.safetensors
- Reboot 1111 with at least a fast SSD
  - There was no dramatic change even if I put it in RamDisk, so as far as I can

## check dependent libraries

- Note that depending on VC++ build tools and CUDA will have a huge impact
- It is ideal to have a VM for operation check separately from the one for development

# Gradio

- Too many things not written in Gradio Docs.

## Blocks

- Basically dig in with a lot of with
- Tab should have no events
- Box does not have a label, the first object's label is used
- all objects are drawn on gradio startup
  - For heavy list display, it is safer to use a mechanism to load after pressing a button

## update()

- Updating objects from the outside just doesn't work
- If you specify it as the output of the event, it will work, but the behavior is limited
- When you come up with an elaborate UI, try it first to see if it can be implemented

## change()

- Overloading causes an infinite loop
- For example, selecting dropdown A changes the contents of dropdown B, and selecting dropdown B puts a value in the textbox.
- That kind of processing pinches the button

##Textbox

- lines are not mutable according to their contents
- Can be resized even with interactive=False
- Strings are unescape()ed arbitrarily. This specification is very troublesome when dealing with paths on Windows
  - A valid workaround is to keep forward slashes on input

## Image

- use change() only when it ends the work flow
- After the upload is complete, the only way for the user to clear it and upload another file is to press the X button
- Big and annoying
- If you return a PIL Image that does not exist as a file, it will not be displayed correctly?
  - Safe to save to a file

##Audio

- Very inconvenient because only one file can be played

## Files

- This is the only way to encourage users to download
- You can't even start the download automatically
- You can't even change the display content
- Make a copy of the file in the tmp folder when processed
  - Can handle multiple files, but be careful as it copies all files

## table view

- gradio does not have a function to display a list with objects
- I will do my best to implement it with table tags, style.css and javascript

## event

- like .click()
- Indentation level is the same as with gr.Blocks()

### _js

- Arguments not documented
- put javascript function name
- called before fn
- The content of return is overwritten by inputs[]
  - If the number of elements in return is insufficient, the value of inputs[] is used

No arguments are allowed, but anonymous functions are allowed.

```python
_js="function(){return rows('"+tab1.lower()+"_"+tab2.lower()+"')}",
```

###fn

Combine getattr and globals if you want to use variables for class and method names.

```python
fn=getattr(globals()[f"FilerGroup{tab1}"], f"download_{tab2.lower()}"),
```

### outputs

- Cannot be specified when empty value becomes None
  - You can return '' to Textbox to display an empty string
  - You cannot clear the image by returning None for Image
