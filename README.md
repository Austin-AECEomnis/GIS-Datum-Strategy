# GIS-Datum-Strategy
Strategy preparing for The North American-Pacific Geopotential Datum of 2022 (NATRF2022), and how to get there from NAD27
# 📐 NAD27 to NAD83(2011) Datum Chain Automation
Scripted Multi-Realization Datum Transformation and Web Map Publication Pipeline — Midland County, TX

## 📋 Overview

Legacy oil and gas land grid data in Texas commonly originates on NAD27, a datum that was never mathematically consistent nationwide since it was built up from ground survey networks over decades rather than a single simultaneous adjustment. Migrating that data to the current NAD83(2011) standard is not a single reprojection step. It is a defined five-hop chain through named intermediate realizations, each requiring its own transformation.

Manually selecting each step in sequence through ArcGIS Pro's interface introduces real risk: a skipped step, a mis-keyed transformation name, or a wrong regional variant can all produce silently incorrect geometry that looks plausible on a map but is wrong underneath.

This repository documents a proof of concept that automates the entire chain as one scripted, auditable arcpy operation, validated against independent authoritative sources, with a second script handling the derived Web Mercator publication layer used for web map display, and a forward-looking design for the eventual NATRF2022 datum transition.

## ⚙️ Requirements

| Requirement | Detail |
|---|---|
| Platform | ArcGIS Pro 3.7.1 (or later, version-matched) |
| Data Component | ArcGIS Coordinate Systems Data, Per User or machine-wide install, version-matched to the installed Pro release |
| Grid Files | NADCON5 CONUS chain confirmed present under `CoordinateSystemsData\pedata\geographic\` |
| Validation Tools | NGS NCAT (web), pyproj (Python library) |

## ⛓️ The Datum Chain

NAD27 was never mathematically consistent across the country. Reaching the current NAD83(2011) standard requires passing through every intermediate realization in order.

| Step | Transformation | What It Is |
|---|---|---|
| 1 | NAD27 to NAD83(1986) | The one true migration. An irregular, grid-based NADCON5 correction. |
| 2 | NAD83(1986) to NAD83(HARN) | State-by-state GPS-based High Accuracy Reference Network readjustments, mostly completed in the 1990s. |
| 3 | NAD83(HARN) to NAD83(FBN) | A further refinement, applying only in states with a second or third HARN readjustment. Not universal. |
| 4 | NAD83(HARN or FBN) to NAD83(NSRS2007) | The 2007 National Re-Adjustment, a simultaneous nationwide GNSS-only adjustment tying all state HARNs to the CORS network. |
| 5 | NAD83(NSRS2007) to NAD83(2011) | The final step to the current standard realization, based on a Multi-Year CORS Solution reprocessing of CORS GPS data. |

Step 1 is the only true migration, since NAD27 itself was never internally consistent. Steps 2 through 5 are reprojections between well-defined, rigorous reference frames. This distinction matters for risk assessment: the NAD27 leg is where irregular, grid-interpolated correction happens, and everything downstream is comparatively well-behaved.

ArcGIS Pro's Project tool interface only allows two transformations to be selected at a time, so this chain cannot be run as a single continuous operation through the dialog. It must either be chained through a bridging datum or, as done here, specified as one composite transformation string in a script.

## 🐍 The Migration Script

```python
import arcpy

in_fc = r"path\to\your.gdb\source_feature_class_nad27"
out_fc = r"path\to\your.gdb\output_feature_class_nad83_2011"

transform_chain = (
    "NAD_1927_To_NAD_1983_7 + "
    "NAD_1983_To_NAD_1983_HARN_47 + "
    "NAD_1983_HARN_To_FBN_NADCON5_3D_CONUS_1 + "
    "NAD_1983_FBN_To_NSRS2007_NADCON5_3D_CONUS_1 + "
    "NAD_1983_NSRS2007_To_2011_NADCON5_3D_CONUS_1"
)

arcpy.management.Project(
    in_dataset=in_fc,
    out_dataset=out_fc,
    out_coor_system=arcpy.SpatialReference(6318),  # NAD 1983 (2011)
    transform_method=transform_chain
)

with arcpy.da.SearchCursor(out_fc, ["SHAPE@"]) as cursor:
    for row in cursor:
        for part in row[0]:
            for pnt in part:
                if pnt:
                    print(f"{pnt.Y:.10f}, {pnt.X:.10f}")
```

This runs the entire five-step chain as a single composite transformation, with no manual selection, no dropdown navigation, and no possibility of skipping a step.

## 🧪 Test Methodology

A single test polygon was digitized in ArcGIS Pro 3.7.1, in a map explicitly set to GCS North American 1927, at a location inside Midland County, Texas, within an active Permian Basin footprint. One vertex coordinate was recorded prior to transformation as ground truth:

```
NAD27 vertex: 31.9047577, -101.9810121
```

The script above was run against the test polygon, producing a NAD83(2011) output feature class. The transformed vertex coordinates were read directly from stored geometry via an arcpy SearchCursor, not from an on-screen hover or map measurement, to eliminate display-time reprojection as a source of ambiguity.

## ✅ Validation

The transformed vertex was independently checked against the NGS NCAT tool (National Geodetic Survey's own authoritative coordinate conversion tool), using the original NAD27 coordinate as input and NAD83(2011) as the requested output.

| Source | Latitude | Longitude |
|---|---|---|
| Script output | 31.9048838850 | -101.9814249050 |
| NGS NCAT | 31.9048838851 | -101.9814248028 |

Agreement to 9 decimal places on latitude and 6 decimal places on longitude, a difference on the order of a hundredth of a millimeter. The composited script reproduces NGS's own authoritative NADCON5 result.

The observed shift between the original NAD27 coordinate and the final NAD83(2011) coordinate is approximately 41 meters, consistent with expected NAD27-to-NAD83 shift magnitudes for this region of Texas.

## 🔧 Known Quirks and Troubleshooting

**1. Grid files not installed by default**
ArcGIS Pro does not ship with the grid files required for NADCON5, NTv2, GEOCON, or similar file-based transformations. These are distributed separately as the ArcGIS Coordinate Systems Data component, downloaded from My Esri and installed independently of Pro itself.

**2. Version matching between Pro and the data component**
The Coordinate Systems Data package must correspond to the installed Pro version. An in-place Pro update patch that runs after the data package is installed can leave the package technically present but effectively mismatched. A full restart of both the application and, if needed, the machine is worth doing before assuming a deeper problem.

**3. Registry path verification**
ArcGIS Pro reads the coordinate systems data location from a registry key at `HKEY_CURRENT_USER\Software\ESRI\Data\CoordinateSystemsData`, value `InstallDir`. Confirming this value points to the actual, correct install location, copy-pasted directly from File Explorer rather than retyped from memory, is a fast way to rule out a path mismatch.

**4. arcpy.ListTransformations() is not reliable for composite NADCON5 chains**
Symptom: `ListTransformations()` consistently returned only legacy, deprecated transformation names such as `NAD_1927_To_NAD_1983_7` and `NAD_1927_To_NAD_1983_NADCON`, never surfacing the modern NADCON5 composite chain, even after the grid files were confirmed present in the correct location with a correctly configured registry pointer. This matches a documented, unresolved pattern reported by other users in Esri's own community forums.

Resolution: Stop relying on `ListTransformations()` for verification. Run the `Project` tool directly with the named composite chain instead. `Project` executed the chain correctly and produced results matching NGS NCAT almost exactly, confirming the enumeration function, not the transformation itself, was the source of the apparent gap.

**5. On-screen measurement is not a reliable validation method**
Symptom: Comparing layers via the Measure tool while the map's display coordinate system remained set to NAD27 produced apparent discrepancies of roughly 40 feet and, separately, 40 meters, in both cases when comparing a transformed output layer against its source.

Cause and resolution: Both discrepancies were attributable entirely to Pro's on-the-fly display reprojection rather than any actual error in the stored data. Validation should always be performed against raw stored coordinates via a SearchCursor, cross-checked against an independent external tool, not against what renders on screen. Once the map's display coordinate system was corrected to match the layers being compared, the visual gap disappeared entirely, confirming the raw coordinate validation had been correct throughout.

## 🔄 The Publication Layer

A second, deliberately separate script handles the NAD83(2011) to Web Mercator (EPSG:3857) projection used for nightly web map refresh:

```python
import arcpy
from datetime import datetime

in_fc = r"path\to\your.gdb\authoritative_nad83_2011"
out_fc = r"path\to\your.gdb\publication_webmercator"

arcpy.management.Project(
    in_dataset=in_fc,
    out_dataset=out_fc,
    out_coor_system=arcpy.SpatialReference(3857)
)

print(f"Web Mercator publication layer regenerated: {datetime.now()}")
```

This is intentionally a separate script from the NAD27 migration chain:

- **Different triggers.** The NAD27 migration is a one-time or rare, backlog-driven event per feature. The Web Mercator projection is a recurring, nightly-cadence operation intended to run indefinitely.
- **Different risk profiles.** The migration script touches authoritative source geometry and requires the validation rigor demonstrated above. The publication script is a pure mathematical projection with no grid-based transformation dependency, low-stakes, and always safely re-runnable from the authoritative source if it fails.
- **Failure isolation.** A failed nightly publication run is a non-event, simply rerun it. A failed migration run touching authoritative data is a real incident requiring investigation. Keeping the scripts separate ensures a fault in one can never cascade into the other.

This mirrors a dual-purpose SDE architecture: NAD83(2011) as the authoritative, editable layer, and Web Mercator as a read-only, script-derived publication layer, regenerated on schedule and never hand-edited.

The Web Mercator output was independently cross-checked against pyproj, a PROJ-based coordinate transformation library run entirely outside of Esri's own projection engine, producing agreement to sub-meter precision. A final visual check, performed only after correcting the map's display coordinate system to match the data being compared, showed no visible gap between the two layers at maximum zoom.

## 🔭 Looking Ahead: NATRF2022

The datum standard itself is scheduled to change again. The North American-Pacific Geopotential Datum of 2022 (NATRF2022) is intended to eventually replace NAD83(2011) as the national standard, tied to modern GNSS reference frames rather than the older Multi-Year CORS Solution basis.

The recommended posture for that eventual transition is a lag strategy, not a lead strategy: remain on NAD83(2011) until primary third-party data vendors, such as parcel data providers and industry data services, confirm their own move to NATRF2022, then convert once in a single clean batch. This avoids a prolonged mixed-frame coexistence window, which is where silent, undetected drift between datasets is most likely to accumulate.

The two scripts in this repository are built to accommodate that future step directly. Once vendor data is confirmed to have moved, the NAD83(2011) to NATRF2022 transition is expected to be a single, well-defined reprojection, appended as a final step once the underlying transformation becomes available in ArcGIS Coordinate Systems Data, rather than a multi-hop chain like the NAD27 migration required.

The non-negotiable control regardless of when that transition happens: every feature should carry enforced CRS and datum metadata at all times. The real risk in any datum transition is not obvious failure, it is silent drift of a few meters between datasets that appear to align but do not.

## 📌 Status

Proof of concept complete and independently validated on real geometry in an active Permian Basin county. Not yet built out to production scale (single-feature batch processing, scheduling, or logging infrastructure). Reserved for future discussion as part of ongoing GIS infrastructure planning.

## 🔗 Related Portfolio Products

- QGIS-LANDFIRE-Pipeline
- ESRI GIS Master Workflow
- ArcGIS Field Maps: Workforce to Tasks

## 👤 Author

**Austin Addington Berlin**
Founder, AECE Omnis LLC
AI-GIS Convergence Research

linkedin.com/in/austinberlin
github.com/Austin-AECEomnis
https://aeceomnis.com/
