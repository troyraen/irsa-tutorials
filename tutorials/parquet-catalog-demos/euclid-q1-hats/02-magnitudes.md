---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.1
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# Euclid Q1 Merged Objects HATS Catalog: Magnitudes

+++

This tutorial explores the different flux measurements available in the Euclid Q1 Merged Objects HATS Catalog and demonstrates how to work with magnitudes in the four Euclid bands.

## Learning Goals

By the end of this tutorial, you will be able to:

- Distinguish between aperture and template-fit flux measurements and understand when to use each type.
- Apply color corrections to NIR band fluxes to obtain accurate total flux estimates.
- Query the HATS Catalog for magnitude data across the four Euclid bands (VIS: I; NISP: Y, J, H).
- Visualize magnitude distributions as a function of object classification.
- Compare aperture and template-fit magnitudes to understand their systematic differences for extended vs. point-like sources.

+++

## 1. Introduction

Euclid Q1 contains two main types of flux measurements: **aperture photometry** and **template-fit photometry**.
These measurements are provided for both Euclid bands (VIS: I; NISP: Y, J, H) and external survey bands.
Additional flux measurements are also available but not covered in this tutorial, including: PSF-fit fluxes (VIS only), Sérsic-fit fluxes (computed for parametric morphology), and fluxes that were corrected based on PHZ classification.

**Aperture fluxes** measure the total light within a defined aperture (e.g., within a Kron radius or a fixed circular aperture).
They are generally more accurate for point-like sources, especially bright stars in the NIR bands, likely due to better handling of PSF-related effects.

**Template-fit fluxes** are expected to be more accurate than aperture fluxes for extended sources because the templates do a better job of excluding contamination from nearby sources.
However, Euclid recommends scaling the measured NIR fluxes with a color term based on VIS photometry to obtain the best estimate of the total flux in each NIR band.
This color correction accounts for systematic differences between the VIS and NIR observations.

See also:
- MER [Photometry Cookbook](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/merdpd/merphotometrycookbook.html)
- [Euclid Collaboration: Romelli et al., 2025](https://arxiv.org/pdf/2503.15305) (hereafter, Romelli)

+++

## 2. Imports

```{code-cell}
# # Uncomment the next line to install dependencies if needed.
# %pip install matplotlib numpy pandas pyarrow s3fs
```

```{code-cell}
import matplotlib.pyplot as plt  # Create figures
import numpy as np  # Math
import pandas as pd  # Manipulate query results
import pyarrow.compute as pc  # Filter dataset
import pyarrow.dataset  # Load the dataset
import pyarrow.fs  # Simple S3 filesystem pointer

# Copy-on-write will become the default in pandas 3.0 and is generally more performant
pd.options.mode.copy_on_write = True
```

## 3. Setup

+++

### 3.1 AWS S3 paths

```{code-cell}
s3_bucket = "nasa-irsa-euclid-q1"
euclid_prefix = "contributed/q1/merged_objects/hats"

euclid_parquet_metadata_path = f"{s3_bucket}/{euclid_prefix}/euclid_q1_merged_objects-hats/dataset/_metadata"

s3_filesystem = pyarrow.fs.S3FileSystem(anonymous=True)
```

### 3.2 Helper functions

The MER Photometry Cookbook defines the magnitude-flux conversion as:

MAG = -2.5 * log10(FLUX) + 23.9

where FLUX is in microjanskys (µJy) and the zeropoint is 23.9.

```{code-cell}
def magnitude_to_flux(magnitude: float) -> float:
    """Convert magnitude to flux following the MER Photometry Cookbook."""
    zeropoint = 23.9
    flux = 10 ** ((magnitude - zeropoint) / -2.5)
    return flux
```

For convenience, we'll have PyArrow compute magnitudes from fluxes during the read operation.
This allows us to avoid loading and handling flux columns directly.
The function below constructs PyArrow expressions that can be used to filter and/or compute new columns on-the-fly.

```{code-cell}
def flux_to_magnitude(flux_col_name: str, color_col_names: tuple[str, str] | None = None) -> pc.Expression:
    """Construct an expression for the magnitude of `flux_col_name` following the MER Photometry Cookbook.

    MAG = -2.5 * log10(TOTAL FLUX) + 23.9.
    If `color_col_names` is None, `flux_col_name` is taken as the total flux in the band.
    If not None, it should list the columns needed for the color correction such that
    TOTAL FLUX = flux_col_name * color_col_names[0] / color_col_names[1].

    Returns a `pyarrow.compute.Expression` which can be used in the `filter` (to filter based on it) and/or
    `columns` (to return it) keyword arguments when loading from the pyarrow dataset.
    """
    scale = pc.scalar(-2.5)
    zeropoint = pc.scalar(23.9)

    total_flux = pc.field(flux_col_name)
    if color_col_names is not None:
        color_scale = pc.divide(pc.field(color_col_names[0]), pc.field(color_col_names[1]))
        total_flux = pc.multiply(total_flux, color_scale)

    log10_flux = pc.log10(total_flux)
    mag_expression = pc.add(pc.multiply(scale, log10_flux), zeropoint)
    return mag_expression
```

### 3.3 Load the catalog as a PyArrow dataset

```{code-cell}
# Load the catalog as a PyArrow dataset. This is used in the examples below.
# Include partitioning="hive" so PyArrow understands the file naming scheme and can navigate the partitions.
dataset = pyarrow.dataset.parquet_dataset(euclid_parquet_metadata_path, partitioning="hive", filesystem=s3_filesystem)
```

### 3.4 Frequently used columns

+++

The following columns are used throughout this tutorial.
Descriptors generally come from Romelli unless noted.

```{code-cell}
# Object ID set by the MER pipeline.
OBJECT_ID = "object_id"

# Whether the source was detected in the VIS mosaic (1) or only in the NIR-stack mosaic (0).
VIS_DET = "mer_vis_det"

# Best estimate of the total flux in the detection band. From aperture photometry within a Kron radius.
# Detection band is VIS if mer_vis_det=1.
# Otherwise, this is a non-physical NIR-stack flux and there was no VIS detection (aka, NIR-only).
FLUX_TOTAL = "mer_flux_detection_total"
FLUXERR_TOTAL = "mer_fluxerr_detection_total"

# Peak surface brightness minus the magnitude used for mer_point_like_prob.
# Point-like: <-2.5. Compact: <-2.6. (Tucci)
MUMAX_MINUS_MAG = "mer_mumax_minus_mag"

# Whether the detection has a >50% probability of being spurious (1=Yes, 0=No).
SPURIOUS_FLAG = "mer_spurious_flag"

# PHZ classification: 1=Star, 2=Galaxy, 4=QSO. Combinations indicate multiple probability thresholds exceeded.
PHZ_CLASS = "phz_phz_classification"
PHZ_CLASS_MAP = {
    1: "Star",
    2: "Galaxy",
    4: "QSO",  # In Q1, this includes luminous AGN.
    # If multiple probability thresholds were exceeded, a combination of classes was reported.
    3: "Star and Galaxy",
    5: "Star and QSO",
    6: "Galaxy and QSO",
    7: "Star, Galaxy, and QSO",
    # Two other integers (-1 and 0) and nulls are present, indicating that the class was not determined.
    **{i: "Undefined" for i in [-1, 0, np.nan]},
}
```

## 4. Magnitude distributions by object type

+++

In this section, we query the catalog for magnitudes in the four Euclid bands and examine their distributions as a function of PHZ classification.
We use template-fit fluxes with color corrections as our baseline, following Euclid's recommendations for extended sources.

Construct the filter for quality objects with VIS detections:

```{code-cell}
# Construct a basic filter.
mag_filter = (
    (pc.field(VIS_DET) == 1)  # No NIR-only objects. For simpler total-flux calculations; not required.
    & (pc.field(PHZ_CLASS).isin([1, 2, 3, 4, 5, 6, 7]))  # Stars, Galaxies, QSOs, and mixed classes.
    & (pc.field(SPURIOUS_FLAG) == 0)  # Basic quality cut.
)
```

Define the magnitude columns we want to load.
Because we're having PyArrow construct new columns (magnitudes from fluxes), we pass a dictionary mapping column names to PyArrow expressions rather than a simple list of column names.

For the VIS band (I), we use `FLUX_TOTAL` which is the best estimate for the total flux in the detection band.
This comes from aperture photometry within a Kron radius.

For the NIR bands (Y, J, H), we compute color-corrected total magnitudes from both template-fit and aperture fluxes.
The color correction scales the measured NIR flux by the ratio of VIS total flux to VIS template (or aperture) flux.

```{code-cell}
# Columns we want to load. Dict instead of list because we're defining new columns (magnitudes).
# We'll have PyArrow return only magnitudes so that we don't have to handle all the flux columns in memory.
_mag_columns = {
    # FLUX_TOTAL is the best estimate for the total flux in the detection band (here, VIS) and comes from
    # aperture photometry. VIS provides the template for NIR bands. It has no unique templfit flux itself.
    "I total": flux_to_magnitude(FLUX_TOTAL),
    # Template-fit fluxes with color corrections.
    "Y templfit total": flux_to_magnitude("mer_flux_y_templfit", color_col_names=(FLUX_TOTAL, "mer_flux_vis_to_y_templfit")),
    "J templfit total": flux_to_magnitude("mer_flux_j_templfit", color_col_names=(FLUX_TOTAL, "mer_flux_vis_to_j_templfit")),
    "H templfit total": flux_to_magnitude("mer_flux_h_templfit", color_col_names=(FLUX_TOTAL, "mer_flux_vis_to_h_templfit")),
    # Aperture fluxes with color corrections.
    "Y aperture total": flux_to_magnitude("mer_flux_y_2fwhm_aper", color_col_names=(FLUX_TOTAL, "mer_flux_vis_2fwhm_aper")),
    "J aperture total": flux_to_magnitude("mer_flux_j_2fwhm_aper", color_col_names=(FLUX_TOTAL, "mer_flux_vis_2fwhm_aper")),
    "H aperture total": flux_to_magnitude("mer_flux_h_2fwhm_aper", color_col_names=(FLUX_TOTAL, "mer_flux_vis_2fwhm_aper")),
}
mag_columns = {**_mag_columns, PHZ_CLASS: pc.field(PHZ_CLASS), MUMAX_MINUS_MAG: pc.field(MUMAX_MINUS_MAG)}
```

Load the data. This query filters for quality VIS-detected objects and returns magnitudes in all four Euclid bands.

```{code-cell}
mags_df = dataset.to_table(columns=mag_columns, filter=mag_filter).to_pandas()
```

Plot magnitude distributions for each band, separated by PHZ classification.
We'll use template-fit magnitudes (with color corrections) as our baseline.
Include multiply-classed objects and separate point-like sources, since classification thresholds were tuned differently for different science goals.

```{code-cell}
# Galaxy + any. Star + galaxy. QSO + galaxy.
classes = {"Galaxy": (2, 3, 6, 7), "Star": (1, 3), "QSO": (4, 6)}
class_colors = ["tab:green", "tab:blue", "tab:orange"]

bands = ["I total", "Y templfit total", "J templfit total", "H templfit total"]
mag_limits = (14, 28)  # Excluding all magnitudes outside this range.
hist_kwargs = dict(bins=20, range=mag_limits, histtype="step")

fig, axes = plt.subplots(3, 4, figsize=(18, 12), sharey="row", sharex=True)
for (class_name, class_ids), class_color in zip(classes.items(), class_colors):
    # Get the objects that are in this class only.
    class_df = mags_df.loc[mags_df[PHZ_CLASS] == class_ids[0]]
    # Plot histograms for each band. Galaxies on top row, then stars, then QSOs.
    axs = axes[0] if class_name == "Galaxy" else (axes[1] if class_name == "Star" else axes[2])
    for ax, band in zip(axs, bands):
        ax.hist(class_df[band], label=class_name, color=class_color, **hist_kwargs)

    # Get the objects that were accepted as multiple classes.
    class_df = mags_df.loc[mags_df[PHZ_CLASS].isin(class_ids)]
    label = "+Galaxy" if class_name != "Galaxy" else "+any"
    # Of those objects, restrict to the ones that are point-like.
    classpt_df = class_df.loc[class_df[MUMAX_MINUS_MAG] < -2.5]
    # Plot histograms for both sets of objects.
    for ax, band in zip(axs, bands):
        ax.hist(class_df[band], color=class_color, label=label, linestyle=":", **hist_kwargs)
        ax.hist(classpt_df[band], color=class_color, linestyle="-.", label=label + " and point-like", **hist_kwargs)

# Add axis labels, etc.
for ax in axes[:, 0]:
    ax.set_ylabel("Counts")
    ax.legend(framealpha=0.2, loc=2)
for axs, band in zip(axes.transpose(), bands):
    axs[0].set_title(band.split()[0])
    axs[-1].set_xlabel(f"{band} (mag)")
plt.tight_layout()
```

Euclid is tuned to detect galaxies for cosmology studies, so there are many more galaxies than other object types.
The green distributions (top row) show the galaxy magnitudes across all four bands.
The green dash-dot line highlights the population of point-like "galaxies", which are likely misclassified stars or QSOs but mostly appear at faint magnitudes.

The star distributions (middle row, blue) are broader and peak at brighter magnitudes than the galaxy distributions, as expected.
Adding objects classified as both star and galaxy (dotted line) adds significant numbers, especially near the peak and toward the faint end where confusion is more likely.
Restricting these to point-like objects (dash-dot line) shows that many bright objects surpassing both probability thresholds are likely true stars.
However, this doesn't hold at the faint end where even some star-only classified objects fail the point-like cut.

The bottom panel (orange) shows very few point-like QSOs, reminding us that most QSO classifications in Q1 should be treated with skepticism.
The few high-confidence QSOs are concentrated in regions with additional external data (particularly u-band from UNIONS in EDF-N).

+++

## 5. Template-fit vs. aperture magnitudes

+++

Now we compare template-fit and aperture magnitudes by plotting their differences.
This comparison reveals systematic offsets that depend on factors including morphology (extended vs. point-like) and brightness.

This figure is inspired by Romelli Fig. 6 (top panel).

```{code-cell}
# Only consider objects within these mag and mag difference limits.
mag_limits, mag_diff_limits = (16, 24), (-1, 1)
mag_limited_df = mags_df.loc[(mags_df["I total"] > mag_limits[0]) & (mags_df["I total"] < mag_limits[1])]

fig, axes = plt.subplots(2, 3, figsize=(18, 9), sharey=True, sharex=True)
bands = [("Y templfit total", "Y aperture total"), ("J templfit total", "J aperture total"), ("H templfit total", "H aperture total")]
hexbin_kwargs = dict(bins="log", extent=(*mag_limits, *mag_diff_limits), gridsize=25)
annotate_kwargs = dict(xycoords="axes fraction", ha="left", fontweight="bold", bbox=dict(facecolor="white", alpha=0.8))

# Plot
for axs, (ref_band, aper_band) in zip(axes.transpose(), bands):
    # Extended objects, top row.
    ax = axs[0]
    extended_df = mags_df.loc[mags_df[MUMAX_MINUS_MAG] >= -2.5]
    extended_mag_diff = extended_df[ref_band] - extended_df[aper_band]
    cb = ax.hexbin(extended_df["I total"], extended_mag_diff, **hexbin_kwargs)
    plt.colorbar(cb)
    ax.set_ylabel(f"{ref_band} - {aper_band}")
    # Annotate top (bottom) with the fraction of objects having a magnitude difference greater (less) than 0.
    frac_tmpl_greater = len(extended_mag_diff.loc[extended_mag_diff > 0]) / len(extended_mag_diff)
    ax.annotate(f"{frac_tmpl_greater:.3f}", xy=(0.01, 0.99), va="top", **annotate_kwargs)
    frac_tmpl_less = len(extended_mag_diff.loc[extended_mag_diff < 0]) / len(extended_mag_diff)
    ax.annotate(f"{frac_tmpl_less:.3f}", xy=(0.01, 0.01), va="bottom", **annotate_kwargs)

    # Point-like objects, bottom row.
    ax = axs[1]
    pointlike_df = mags_df.loc[mags_df[MUMAX_MINUS_MAG] < -2.5]
    pointlike_mag_diff = pointlike_df[ref_band] - pointlike_df[aper_band]
    cb = ax.hexbin(pointlike_df["I total"], pointlike_mag_diff, **hexbin_kwargs)
    plt.colorbar(cb)
    ax.set_ylabel(f"{ref_band} - {aper_band}")
    # Annotate top (bottom) with the fraction of objects having a magnitude difference greater (less) than 0.
    frac_tmpl_greater = len(pointlike_mag_diff.loc[pointlike_mag_diff > 0]) / len(pointlike_mag_diff)
    ax.annotate(f"{frac_tmpl_greater:.3f}", xy=(0.01, 0.99), va="top", **annotate_kwargs)
    frac_tmpl_less = len(pointlike_mag_diff.loc[pointlike_mag_diff < 0]) / len(pointlike_mag_diff)
    ax.annotate(f"{frac_tmpl_less:.3f}", xy=(0.01, 0.01), va="bottom", **annotate_kwargs)

# Add axis labels, etc.
for i, ax in enumerate(axes.flatten()):
    ax.axhline(0, color="gray", linewidth=1)
    if i == 1:
        ax.set_title("Extended objects")
    if i == 4:
        ax.set_title("Point-like objects")
    if i > 2:
        ax.set_xlabel("I total")
plt.tight_layout()
```

The template - aperture magnitude difference is fairly tightly clustered around 0 for extended objects (top row), but with asymmetric outliers.
Fractions annotated in the plots show the proportion of objects with magnitude differences above (top number) and below (bottom number) zero.
There is a positive offset indicating fainter template-fit magnitudes, as expected: templates better exclude contaminating light from nearby sources.

The offset is more pronounced for point-like objects (bottom row), likely due to PSF-related issues in template fitting.
This confirms that aperture magnitudes are more reliable for point sources, consistent with recommendations in the MER Photometry Cookbook.

+++

## About this notebook

**Authors:** Troy Raen (Developer; Caltech/IPAC-IRSA) and the IRSA Data Science Team.

**Updated:** 2025-12-10

**Contact:** [IRSA Helpdesk](https://irsa.ipac.caltech.edu/docs/help_desk.html) with questions or problems.
