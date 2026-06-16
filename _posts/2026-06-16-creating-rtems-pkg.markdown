---
layout: post
title:  "Building rtems-pkg: A Universal Packaging Wrapper"
date:   2026-06-16 09:30:00 +0530
categories: jekyll update
---

> Love is the death of duty.

Morning ladies and gentlemen.

With the temperatures here in India reaching 43°C during the day, I realised I am made for the
North which knows no king but the king in the North, whose name is Stark! ;|

Moving ahead we are close to the GSoC mid-term evaluation. I have almost completed all my mid-term
goals.

After fixing the RPM packaging pipeline and building the Debian one from scratch, 
I ran into a problem: almost every packager has a different packaging flow, and 
remembering all of them is a pain. What I really wanted was a single command that 
could switch between packagers on the fly, without having to memorize a dozen 
different invocations for each one.
 
So I decided to write a wrapper in Python.
 
## The vision
 
An abstraction layer that wraps other packagers and their individual flows into 
a single, intuitive CLI.
 
```
./rtems-pkg --packager rpm --target amd/amd-kria-k26
```
 
where,
```
--packager  Choose the packaging format to build
--target    TARGET Board to build
```
 
## The planning
 
Here's where it gets interesting. I had a rough vision for how this wrapper should 
be structured, drawing from my experience writing Discord bots most of which are 
organized so that each command is its own module. I wanted the same here: each 
packager would be a self-contained module, making it easy to add new packaging 
formats without touching the existing code.
 
To start, I defined a base `Packager` class:
 
```python
class Packager(ABC):
 
    def __init__(self, name: str):
        self.name = name
 
    def log(self, msg: str, level: str = "info"):
        message = f"[{self.name.upper()}] {msg}"
        if level == "error":
            logging.error(message)
        else:
            logging.info(message)
 
    @abstractmethod
    def get_build_config(self, target: str) -> dict:
        """Subclasses must return the required path, command, and working directory."""
        pass
 
    def check_if_packager_exists(self, cmd):
        """Check if the required packaging tool is available in the system."""
        tool_name = cmd[0]
        if shutil.which(tool_name) is None:
            self.log(f"Required tool '{tool_name}' not found in PATH.",
                     level="error")
            raise PackagerError(
                f"{tool_name} is not installed or not in PATH.")
 
    def run(self, target: str, board_name: str):
        config = self.get_build_config(target)
        target_path = config['path']
        cmd = config['cmd']
        cwd = config.get('cwd')  # Optional working directory
 
        self.check_if_packager_exists(cmd)
 
        if not os.path.exists(target_path):
            self.log(f"Generated target not found: {target_path}",
                     level="error")
            raise FileNotFoundError(f"Missing required path: {target_path}")
 
        self.log(f"Building package for {board_name} from {target_path}...")
        try:
            subprocess.run(cmd, cwd=cwd, check=True)
            self.log(f"SUCCESS: Package built successfully for {board_name}!")
        except subprocess.CalledProcessError as e:
            self.log(f"ERROR: Build failed with exit code {e.returncode}",
                     level="error")
            raise PackagerError(f"{self.name} build failed") from e
```
 
This is the base class that other packager classes inherit from. Most of the methods
are pretty self-explanatory. It also validates that the required packaging tool 
actually exists on the host system before attempting a build, via `check_if_packager_exists`.
 
The heart of this class is the `run` method that's where all the magic happens.
 
Any packager inheriting from this class needs to implement `get_build_config()`, 
returning a dictionary with these fields:
 
```python
def get_build_config(self, target: str) -> dict:
    spec_file = os.path.join('out', f"{target}.spec")
    return {
        'path': spec_file,
        'cmd': ['rpmbuild', '-bb', spec_file],
        'cwd': None  # rpmbuild can run from anywhere
    }
```
 
Given a target board string, this method returns the required config file, the build command
, and the directory the build command should run in.
 
Tools like `rpmbuild`, which can run from any directory as long as you point them 
at a config file (like a `.spec` file), can skip `cwd` entirely just pass the 
config file via `path`.
 
Other tools, like `dpkg-buildpackage`, need to run from a specific directory and don't
strictly need a separate config file. For these, you can just pass the working 
directory as `path` instead. Since `run()` checks that `path` exists before doing 
anything, it doubles as a sanity check that the directory you're about to `cd` 
into actually exists.
 
Here is an example of a Debian packager:
 
```python
class DEB(Packager):
 
    def __init__(self):
        super().__init__("deb")
 
    def get_build_config(self, target: str) -> dict:
        deb_dir = os.path.join('out', f"{target}.debian")
        return {
            'path': deb_dir,
            'cmd': ['dpkg-buildpackage', '-b', '-uc', '-us'],
            'cwd': deb_dir  # dpkg needs to run inside the directory
        }
```
 
With that in place, the next step was a central registry to keep track of these 
packager modules.
 
I introduced a `PackagerFactory` class where all packager modules get registered. The
value of the `--packager` flag is looked up against this map, and if a match is found,
an instance of the corresponding packager class is created.
 
```python
class PackagerFactory:
    _packagers = {
        # format_type: (waf_feature, packager_class)
        'deb': ["deb", DEB],
        'rpm': ["rpmspec", RPM]
    }
 
    @classmethod
    def get_packager(cls, format_type: str) -> Packager:
        packager_class = cls._packagers.get(format_type.lower())
        if not packager_class:
            raise ValueError(f"Unsupported package format: {format_type}")
        return packager_class[1]()
 
    @classmethod
    def get_waf_target(cls, format_type: str) -> str:
        packager_info = cls._packagers.get(format_type.lower())
        if not packager_info:
            raise ValueError(f"Unsupported package format: {format_type}")
        return packager_info[0]
```
 
The map also stores the corresponding `waf` target, which is responsible for generating
the actual configuration and metadata files from templates.
 
Once the factory hands back the right packager object, building a package is as simple 
as calling `.run()` on it.
 
Here's a quick look at the overall packaging flow we've established so far:
 
```text
rtems-pkg
│
├── Parse CLI args
│     │
│     ├── --packager deb
│     └── --target <board>
│              (e.g. amd/amd-kria-k26, beagle/black)
│
├── Determine waf command
│     │
│     └── PackagerFactory.get_waf_target("deb")
│            │
│            └── returns → "deb"
│                         or "rpm"
│
├── Generate packaging configs for ALL boards
│     │
│     └── ./waf deb
│             │
│             ├── creates Debian metadata
│             │      ├── debian/
│             │      ├── control files
│             │      └── per-board package configs
│             │             
│             └── OR ./waf rpm
│                    └── generates rpm metadata(.spec)
│
├── Create correct packager object
│     │
│     └── PackagerFactory.get_packager("deb")
│            │
│            └── returns → DebPackager()
│
└── Build package for requested target
      │
      └── packager.run(target)
              │
              ├── target = value from --target
              │
              └── selects board-specific config
                   and builds package for it
```
 
## Adding new packagers
 
Once you understand the underlying architecture, adding a new packager is fairly straightforward.
 
**First and foremost**, you'd need to add a new target to `waf` I'll cover how to do that
in a later post. You'd also need to add templates for the relevant metadata files under
the `pkg/` directory.
 
To wire it into `rtems-pkg`, create a new class inheriting from `Packager`:
 
```python
class MY_PACKAGER(Packager):
    def __init__(self):
        super().__init__("my-pkgr")
 
    def get_build_config(self, target: str) -> dict:
        metadata_file = os.path.join('out', f"{target}.buildconfig")
        # ^^^ READ THE NOTE BELOW ^^^
        return {
            'path': metadata_file,
            'cmd': ['my-pkgr', '--please', '--package-this', metadata_file],
            'cwd': None  # assuming this can run from anywhere
        }
```
 
> [!NOTE]
> Each packager class is solely responsible for figuring out the exact metadata file or directory
> it needs to run from. The information below should help.
 
The `--target` argument in `rtems-pkg` is essentially a path mirroring the board
hierarchy `<vendor>/<board>`:
 
```
--target amd/amd-kria-k26
--target beagle/black
```
 
`waf` uses this same path to name whatever it generates under `out/`, following the pattern 
`<vendor>/<board>.<packager-extension>`. The extension (or directory suffix) depends 
entirely on which packager produced it `waf rpm` writes a `.spec` file, `waf deb` writes 
a `.debian/` directory, and a new packager would follow the same convention with its own 
suffix. For a hypothetical packager called `somenewpkgr`, that might look like
`out/amd/amd-kria-k26.somenewpkgr`:
 
```text
out/
├── amd/
│   ├── amd-kria-k26.spec          # generated by waf rpm
│   ├── amd-kria-k26.debian/       # generated by waf deb
│   └── amd-kria-k26.somenewpkgr   # generated by waf somenewpkgr
│
└── beagle/
    ├── black.spec
    └── black.debian/
```
 
This is exactly why `get_build_config()` is left to each packager subclass to implement only
that packager knows what suffix or directory structure to look for, given the `--target` 
string. It just joins `'out'`, the target, and its own extension, the same way the `RPM`
and `DEB` examples do above.
 
Once the new packager class is ready, register it in `PackagerFactory`:
 
```python
class PackagerFactory:
    _packagers = {
        # format_type: (waf_feature, packager_class)
        'deb': ["deb", DEB],
        'rpm': ["rpmspec", RPM],
        'mypkg': ["mypkg", MY_PACKAGER]
    }
```
 
That's it we've successfully added a new packager.
 
Running:
```
./rtems-pkg --packager mypkg --target amd/amd-kria-k26
```
 
will now generate the `mypkg` metadata via `waf` and then invoke the packager executable.

In the next blog, I would be implementing FreeBSD packaging support. 

Until then Sayounara!
