# GuiggianiRichardsonDuffy Integration

This directory (`src/guiggiani`) is a junction/symlink pointing to the local clone of GuiggianiRichardsonDuffy.

**Local path:** `C:\Users\Adrien VET\Documents\Thèse\julia_venv\GuiggianiRichardsonDuffy`

**GitHub repository:** https://github.com/Aguelord/GuiggianiRichardsonDuffy

## Setup Instructions

If you clone this fork of Inti.jl, you need to:

1. Clone GuiggianiRichardsonDuffy separately
2. Create a junction: 
   ```
   cmd /c mklink /J "src\guiggiani" "path\to\GuiggianiRichardsonDuffy"
   ```

## Development Workflow

- Modify GRD in its separate repository
- Changes are automatically available in Inti via the junction
- Commit and push GRD changes from the GRD repository
