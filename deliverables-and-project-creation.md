# Deliverables & Auto Project Creation

## 1. Deliverables Tab

### Overview

The Deliverables tab is the primary interface for capturing what needs to be **installed on-site**, rather than diving straight into material specifications. Each deliverable represents a distinct physical component of a project (e.g. fascia sign, hanging sign, vehicle wrap).

### Capturing Deliverables from Survey Data

Survey photos are captured on-site with associated **height** and **width** dimensions stored in the database. When creating a deliverable, these dimensions auto-populate from the linked survey record — no manual re-entry required.

### Material Integration

Materials already in the database are tagged by type (substrate, vinyl, laminate). When a deliverable is defined, the user selects the appropriate material type. This drives both the cutting calculations and the material order quantities.

### Cutting Diagram Generation

Once a deliverable has its dimensions and material type confirmed, the user can generate a cutting diagram via a single button press. The system:

- Calculates the most efficient way to cut the required dimensions from standard sheet sizes
- Accounts for whether the sign needs to be pieced together from multiple sheets or cut down from a larger sheet
- Exports the diagram as an **SVG or PNG**
- Saves it automatically to the **Links folder** of the corresponding Google Drive project folder

The design team can then reference this file directly in Illustrator, giving them the physical constraints before they begin laying out artwork.

---

## 2. Auto Project Creation & Google Drive Integration

### Project Folder Modal

When creating a new project in VisualOS, the system **prompts the user** with a simple modal before any folder is created or linked:

> **"Does a Google Drive folder already exist for this project?"**
>
> [ 🔗 Link Existing Folder ]   [ 📁 Create New Folder ]

The modal has two buttons only — keep it simple.

### Option A: Link Existing Folder

Clicking **Link Existing Folder** opens the **Google Drive Picker**, allowing the user to navigate to and select the existing project folder. Once selected, that folder is linked to the project record in VisualOS.

### Option B: Create New Folder

Clicking **Create New Folder** triggers the automated folder creation process:

1. Creates a new folder in the correct hierarchy: `Letter / Client Name / Year / Job Number`
2. Copies the **industry-standard base template AI file** from the designated template folder in Google Drive into the new project folder
3. **Renames the AI file** to match the job name

The project record in VisualOS is then automatically linked to the new folder.

---

## Notes

- The cutting diagram skill/calculator is a reusable module that accepts dimensions + material type and returns optimised cut layouts based on standard sheet sizes already stored in the system.
- The base template AI file is universal across all industry types (signage, vehicle, apparel, etc.).
- Future enhancement: explore embedding cutting diagrams directly into the AI file artboard via UXP scripting.
