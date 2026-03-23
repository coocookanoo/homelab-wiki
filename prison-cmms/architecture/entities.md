# Entities & Data Model

## Entity Hierarchy

```
Zone
 └── Cell
      ├── Door
      └── Camera

Staff ──► WorkOrder ──► AssetCategory
```

## Entities

### Zone
Represents a security area within the facility (e.g., Cell Block A, Admin Wing).

### Cell
A specific room/cell within a Zone. Tracks inmate assignment and capacity.

### Door
A security door access point within a Cell/area.

### Camera
A surveillance camera and its location within a Cell/area.

### Staff
Personnel records — name, role, badge number.

### WorkOrder
Maintenance task assigned to Staff. Links to an AssetCategory.
- Supports AI-generated maintenance suggestions
- Tracks status: open, in-progress, closed

### AssetCategory
One of 14 CMMS facility maintenance categories with optional subcategories.
See [[architecture/asset-hierarchy]] for the full list.

### User
Authentication user. Separate from Staff.

## Key Relationships

- A Zone has many Cells
- A Cell belongs to one Zone
- Doors and Cameras belong to a Cell
- WorkOrders are assigned to Staff
- WorkOrders reference an AssetCategory

## See Also

- [[architecture/api-patterns]] — URL conventions for each entity
- [[architecture/asset-hierarchy]] — Asset category details
