# icontool

A command-line tool for reading and modifying Amiga Workbench `.info` icon files.

## Features

- **Inspect** icon metadata, structure fields, and ToolTypes
- **Edit** any structure field using path notation (e.g., `DiskObject:Gadget:LeftEdge=100`)
- **Modify** ToolTypes with key-based operations (set, get, delete, toggle flags)
- **Export** embedded icons to PNG files
- **Import** PNG files as icon graphics (with automatic palette quantization)
- **Batch process** multiple files in a single command
- **Preserve** GlowIcons/NewIcons data
- **Non-destructive** workflow with separate output file option

## Installation

icontool requires Python 3.7+. No dependencies are required for basic operations.

For PNG import/export, install pypng:

```bash
pip install pypng
```

## Usage

```
icontool [options] <file.info> [file2.info ...]
```

Files can be specified with or without the `.info` extension.

## Inspecting Icons

### Basic listing

```bash
icontool --list MyIcon.info
```

Output:
```
MyIcon.info:
  type: 3 (tool)
  version: 1, revision: 0, main_icons: 2
  currentxy: 100,50
  canvas: 48x32
  has_drawerdata: False  has_defaulttool: True  has_tooltypes: True
  DONOTWAIT
  TOOLPRI=0
```

### Verbose structure dump

```bash
icontool --list -v MyIcon.info
```

Displays all structure fields with path notation:
```
MyIcon.info:
DiskObject:Magic=0xE310 (valid)
DiskObject:Version=1
DiskObject:Type=3 (WBTOOL)
DiskObject:CurrentX=100
DiskObject:CurrentY=50
DiskObject:StackSize=4096
DiskObject:Gadget:LeftEdge=0
DiskObject:Gadget:TopEdge=0
DiskObject:Gadget:Width=48
DiskObject:Gadget:Height=32
DiskObject:Gadget:Flags=0x5
DiskObject:Gadget:UserData=0 (OS1.x)
Icon:Width=48
Icon:Height=32
Icon:Depth=2
IconSelect:Width=48
IconSelect:Height=32
IconSelect:Depth=2
DefaultTool="SYS:Tools/IconEdit"
ToolTypes[0]="DONOTWAIT"
ToolTypes[1]="TOOLPRI=0"
```

## Editing Structure Fields

Use `--edit` with path notation to modify any field:

```bash
# Set icon position
icontool --edit "DiskObject:CurrentX=200" --edit "DiskObject:CurrentY=100" MyIcon.info

# Change stack size
icontool --edit "DiskObject:StackSize=8192" MyIcon.info

# Modify gadget dimensions
icontool --edit "DiskObject:Gadget:Width=64" --edit "DiskObject:Gadget:Height=48" MyIcon.info
```

### Available field paths

**DiskObject** (main header):
- `DiskObject:Magic` - Magic number (0xE310)
- `DiskObject:Version` - Structure version
- `DiskObject:Type` - Icon type (1=disk, 2=drawer, 3=tool, 4=project, etc.)
- `DiskObject:CurrentX`, `DiskObject:CurrentY` - Icon position
- `DiskObject:StackSize` - Stack size for tools
- `DiskObject:DefaultTool`, `DiskObject:ToolTypes` - Presence pointers
- `DiskObject:DrawerData`, `DiskObject:ToolWindow` - Presence pointers

**DiskObject:Gadget** (embedded gadget):
- `DiskObject:Gadget:LeftEdge`, `TopEdge` - Position within gadget
- `DiskObject:Gadget:Width`, `Height` - Gadget dimensions
- `DiskObject:Gadget:Flags`, `Activation`, `GadgetType`
- `DiskObject:Gadget:GadgetRender`, `SelectRender` - Image pointers
- `DiskObject:Gadget:UserData` - OS version indicator (0=OS1.x, 1=OS2.x)

**DrawerData:NewWindow** (for disk/drawer icons):
- `DrawerData:NewWindow:LeftEdge`, `TopEdge`, `Width`, `Height`
- `DrawerData:NewWindow:MinWidth`, `MinHeight`, `MaxWidth`, `MaxHeight`
- `DrawerData:NewWindow:DetailPen`, `BlockPen`
- `DrawerData:NewWindow:Flags`, `IDCMPFlags`, `Type`

**DrawerData**:
- `DrawerData:CurrentX`, `DrawerData:CurrentY`

**Icon** / **IconSelect** (image headers):
- `Icon:LeftEdge`, `TopEdge`, `Width`, `Height`, `Depth`
- `Icon:PlanePick`, `PlaneOnOff`
- `IconSelect:*` - Same fields for select (highlighted) icon

## ToolTypes Operations

### Get a ToolType value

```bash
icontool --get PUBSCREEN MyIcon.info
# Output: MyIcon.info: PUBSCREEN=Workbench

icontool --get DONOTWAIT MyIcon.info
# Output: MyIcon.info: DONOTWAIT=true
```

### Set a key=value ToolType

```bash
icontool --set "PUBSCREEN=MyScreen" MyIcon.info
```

### Set a boolean flag (enabled)

```bash
icontool --set-flag DONOTWAIT MyIcon.info
```

### Unset a boolean flag (disabled)

Writes `(FLAGNAME)` to disable without removing:

```bash
icontool --unset DONOTWAIT MyIcon.info
```

### Delete a ToolType entirely

Removes all forms (KEY, KEY=value, and (KEY)):

```bash
icontool --delete PUBSCREEN MyIcon.info
```

### Case-insensitive matching

```bash
icontool --get pubscreen --ignore-case MyIcon.info
```

### Normalize ToolTypes

Remove duplicates (last value wins) and empty entries:

```bash
icontool --normalize MyIcon.info
```

## Icon Position and Window Size

### Set icon position

```bash
icontool --pos 100,200 MyIcon.info
```

### Set drawer window size

For disk/drawer icons only:

```bash
icontool --winsize 640,480 MyIcon.info
```

## PNG Export

Export embedded icons to PNG files (requires pypng):

```bash
# Export main icon
icontool --export-icon icon.png MyIcon.info

# Export select (highlighted) icon
icontool --export-select-icon icon_select.png MyIcon.info

# Export both
icontool --export-icon main.png --export-select-icon select.png MyIcon.info
```

Icons are exported using the appropriate Workbench palette:
- **WB 1.3**: Blue, white, black, orange + 4 additional colors
- **WB 2.0+**: Gray, black, white, blue, red, green, dark blue, orange

## PNG Import

Import PNG files as icon graphics (requires pypng):

```bash
# Replace main icon
icontool --import-icon newicon.png MyIcon.info

# Replace select icon
icontool --import-select-icon newicon_sel.png MyIcon.info
```

The import process:
1. Reads the PNG file
2. Quantizes colors to both WB1.3 and WB2.0 palettes
3. Selects the palette with lower color error
4. Determines minimum bit depth needed (1-3 planes)
5. Updates gadget dimensions if the new icon is larger

## Output Options

### Separate output file (non-destructive)

```bash
icontool --edit "DiskObject:StackSize=8192" -o modified.info original.info
```

### Dry run (preview changes)

```bash
icontool --edit "DiskObject:CurrentX=100" --dry-run MyIcon.info
# Output: MyIcon.info: would update
```

### Create backup before modifying

```bash
icontool --edit "DiskObject:CurrentX=100" --backup MyIcon.info
# Creates MyIcon.info.bak (once, on first modification)
```

### Compact file structure

Rebuilds the file to remove gaps in ToolTypes table:

```bash
icontool --compact MyIcon.info
```

### Strict validation

Fail fast on suspicious data:

```bash
icontool --strict --list MyIcon.info
```

## Batch Processing

Process multiple files in one command:

```bash
# Set position on all icons
icontool --pos 50,50 *.info

# List all icons
icontool --list Icons/*.info

# Normalize ToolTypes across project
icontool --normalize --backup Project/#?.info
```

Note: `--output` cannot be used with multiple input files.

## Examples

### Create a tool icon workflow

```bash
# Start with existing icon, modify for new tool
icontool \
  --edit "DiskObject:StackSize=16384" \
  --set "PUBSCREEN=Workbench" \
  --set-flag DONOTWAIT \
  --delete TOOLPRI \
  --import-icon mytool.png \
  -o MyTool.info \
  Template.info
```

### Inspect and fix icon positions

```bash
# Check current positions
icontool --list *.info | grep currentxy

# Reset all to NO_ICON_POSITION
icontool --pos 0x80000000,0x80000000 *.info
```

### Export icons for editing

```bash
# Export, edit in image editor, re-import
icontool --export-icon temp.png MyIcon.info
# ... edit temp.png ...
icontool --import-icon temp.png MyIcon.info
```

## File Format Notes

icontool handles standard Amiga `.info` files including:

- **DiskObject** header (78 bytes)
- **DrawerData** for disk/drawer icons (56 bytes + OS2.x extension)
- **Image** structures with planar bitmap data
- **DefaultTool** string
- **ToolTypes** string array
- **GlowIcons** / NewIcons data (preserved but not modified)

Icon types supported:
| Type | Value | Description |
|------|-------|-------------|
| WBDISK | 1 | Disk/volume icon |
| WBDRAWER | 2 | Drawer/directory icon |
| WBTOOL | 3 | Executable tool |
| WBPROJECT | 4 | Project/document |
| WBGARBAGE | 5 | Trashcan |
| WBDEVICE | 6 | Device icon |
| WBKICK | 7 | Kickstart disk |
| WBAPPICON | 8 | Application icon |

## Exit Codes

- **0** - Success
- **1** - Partial failure (some operations failed)
- **2** - File read/parse error

## License

See LICENSE file in the repository.
