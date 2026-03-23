# Asset Hierarchy

The system includes 14 top-level CMMS asset categories with subcategories (84 total).

## Top-Level Categories

| # | Category | Description |
|---|---|---|
| 1 | HVAC | Heating, ventilation, air conditioning |
| 2 | Electrical | Power systems, lighting, panels |
| 3 | Plumbing | Water supply, drainage, fixtures |
| 4 | Fire & Life Safety | Sprinklers, alarms, extinguishers |
| 5 | Security | Access control, perimeter security |
| 6 | Surveillance | Cameras, recording systems, monitoring |
| 7 | Structural | Building envelope, roofing, foundations |
| 8 | Grounds | Exterior areas, landscaping, fencing |
| 9 | Elevators | Lifts, escalators, vertical transport |
| 10 | Environmental | Air quality, water treatment, waste |
| 11 | Operations | General operations, janitorial, equipment |
| 12 | Emergency Response | Emergency systems, backup power |
| 13 | Predictive Maintenance | Condition monitoring, inspections |
| 14 | Administrative | Office equipment, IT, communications |

## Seeding

Run the init script to populate the database:

```bash
python init_asset_hierarchy.py
```

The script is idempotent — safe to run multiple times.

## API

```bash
# List all top-level categories
GET /asset-categories/

# Get category with subcategories
GET /asset-categories/{id}
```

## See Also

- [[features/work-orders]] — How work orders use asset categories
