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

# Euclid Q1 Merged Objects HATS Catalog: Introduction

+++

This tutorial is an introduction to the Euclid Q1 Merged Objects HATS Catalog.
It is the first in a series

## Learning Goals

By the end of this tutorial series, you will be able to:

- Understand the format and organization of the Euclid Q1 Merged Objects HATS Catalog.
- Distinguish between the different redshift estimates, different flux measurements, etc. and choose appropriate options for your use case.
- Query the HATS Catalog for quality samples of galaxies, QSOs, stars, and NIR-only objects and plot distributions and measurement comparisons to better understand the data.

In this introductory tutorial, you will:

- Learn about the 14 Euclid Q1 tables that were merged to create the HATS Catalog and know where to look for more information about the algorithms that produced them.
- Understand the format and organization of the HATS Catalog.
- Be able to find columns of interest, both in the HATS Catalog schema and the Euclid project documentation.

+++

## 1. Introduction

The [Euclid Q1](https://irsa.ipac.caltech.edu/data/Euclid/docs/overview_q1.html) catalogs were derived from Euclid photometry and spectroscopy, taken by the Visible Camera (VIS) and the Near-Infrared Spectrometer and Photometer (NISP), and from photometry taken by other ground-based instruments.
The data include several flux measurements per band, several redshift estimates, several morphology parameters, etc.
Each was derived for different science goals using different algorithms or configurations.

The Euclid Q1 Merged Objects HATS Catalog was produced by joining 14 of the original catalogs on object ID (column: `object_id`).
Following the HATS framework, the data were then partitioned spatially (by right ascension and declination) and written as an Apache Parquet dataset.
A visualization of the Euclid Q1 on-sky density and the HATS partitioning of this catalog can be seen on these [skymaps](https://irsa.ipac.caltech.edu/data/download/parquet/euclid/q1/merged_objects/hats/euclid_q1_merged_objects-hats/skymap.png).

- Columns: 1,594
- Rows: 29,953,430 (one per Euclid Q1 object)
- Size: 400 GB

HATS and Parquet are described in more detail at [IRSA Parquet Catalogs](https://irsa.ipac.caltech.edu/docs/parquet_catalogs/).
Both are good for most use cases and especially well-suited for medium- and large-scale analyses.
The catalog is served from an AWS S3 cloud storage bucket.
Access is free and no credentials are required.
S3 efficiently supports both single stream and massively parallel workflows.

### 1.1 Tutorials

This tutorial describes the Euclid catalogs that were merged to create the HATS Catalog and how the HATS Catalog is organized.
It then demonstrates how to load the catalog's schema, find columns of interest, and perform a basic query.

Later tutorials in this series will describe the different redshift estimates, show how to obtain quality samples, and visualize their distributions.

This tutorial series uses the [Apache PyArrow](https://arrow.apache.org/docs/python/index.html) library to work with the HATS Catalog.
PyArrow is a particularly efficient Python library for working with Parquet datasets.
It is also powerful and flexible, supporting both basic and complex SQL-like queries.

LSDB is a Python library developed specifically for HATS.
The HATS Catalog is part of a HATS Collection that also includes a 10 arcsecond Margin Cache and an Index Table that maps `object_id` to the HATS partition.
LSDB natively supports the full Collection and is especially useful for cone searches and cross matching.
See the tutorial [Access HATS Collections Using LSDB: Euclid Q1 and ZTF DR23](../irsa-hats-with-lsdb.md).

The original Euclid Q1 catalogs are also available via IRSA's TAP service, as demonstrated in the [Euclid Q1 tutorial series](FIXME/link).
The performance of small- to medium-scale queries is expected to be roughly similar whether the TAP service or the HATS Catalog is used.

+++

## 2. Imports

```{code-cell}
# # Uncomment the next line to install dependencies if needed.
# %pip install pyarrow s3fs
```

```{code-cell}
import pyarrow.parquet  # Load the schema
import pyarrow.fs  # Simple S3 filesystem pointer
```

## 3. HATS Catalog Contents

+++

The HATS Catalog contains 14 Euclid Q1 catalogs joined together on the column `object_id`.
This section describes the content and column naming conventions shows how to find columns of interest.

See also:
- MER [Photometry](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/merdpd/merphotometrycookbook.html) and [Morphology](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/merdpd/mermorphologycookbook.html) Cookbooks
- [Frequently Asked Questions About Euclid Q1 data](https://euclid.caltech.edu/page/euclid-q1-data-faq) (hereafter, FAQ)
- [Q1 Explanatory Supplement](https://euclid.esac.esa.int/dr/q1/expsup/)

### 3.1 Load the schema (column descriptions)

We follow IRSA's
[Cloud Access tutorial](https://caltech-ipac.github.io/irsa-tutorials/tutorials/cloud_access/cloud-access-intro.html#navigate-a-catalog-and-perform-a-basic-query)
to inspect the catalog's schema, which contains the column definitions.
The schema is accessible from a few locations.
Here, we load it from the `_common_metadata` file because that file includes column metadata (units and descriptions) which is not present elsewhere.

Define the AWS S3 paths:

```{code-cell}
s3_bucket = "nasa-irsa-euclid-q1"
euclid_prefix = "contributed/q1/merged_objects/hats"

euclid_parquet_metadata_path = f"{s3_bucket}/{euclid_prefix}/euclid_q1_merged_objects-hats/dataset/_metadata"
euclid_parquet_schema_path = f"{s3_bucket}/{euclid_prefix}/euclid_q1_merged_objects-hats/dataset/_common_metadata"

s3_filesystem = pyarrow.fs.S3FileSystem(anonymous=True)
```

Load the schema:

```{code-cell}
schema = pyarrow.parquet.read_schema(euclid_parquet_schema_path, filesystem=s3_filesystem)
```

There are almost 1600 columns in this dataset.

```{code-cell}
print(f"{len(schema)} columns total")
```

### 3.2 Column naming conventions

The column names seen at the schema URL for each Euclid table listed in the sections below are the original Euclid names are mostly in all caps.
In the HATS Catalog and the catalogs available through IRSA's TAP service, all column names have been lower-cased.
In addition, all non-alphanumeric characters in column names have been replaced with an underscore for compatibility with various libraries and services.
In the HATS Catalog, the table name has also been prepended (e.g., `E(B-V)` -> `physparamqso_e_b_v_`).

Three columns have special names that differ from the standard naming convention described above:

- `object_id` : Euclid MER Object ID. Unique identifier of a row in this dataset. No table name prepended.
- `ra` : 'RIGHT_ASCENSION' from the 'mer' table. Name shortened to match other IRSA services. No table name prepended.
- `dec` : 'DECLINATION' from the 'mer' table. Name shortened to match other IRSA services. No table name prepended.

### 3.3 MER tables

The Euclid MER processing function produced three tables.
The reference paper is [Euclid Collaboration: Romelli et al., 2025](https://arxiv.org/pdf/2503.15305) (hereafter, Romelli).
The tables, original schemas, and example column name transformations are as follows.

**Main table (mer)**
- Described in Romelli Sections 6 & 8 ("EUC_MER_FINAL-CAT")
- Original schema: [Main catalog FITS file](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/merdpd/dpcards/mer_finalcatalog.html#main-catalog-fits-file)
- Example column name: `FLUX_DETECTION_TOTAL` --> `mer_flux_detection_total`

**Morphology (morph)**
- Described in Romelli Sec. 7 & 8 ("EUC_MER_FINAL-MORPH-CAT")
- Original schema: [Morphology catalog FITS file](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/merdpd/dpcards/mer_finalcatalog.html#morphology-catalog-fits-file)
- Example column name: `CONCENTRATION` --> `morph_concentration`

**Cutouts (cutouts)**
- Described in Romelli Sec. 8 ("EUC_MER_FINAL-CUTOUTS-CAT")
- Original schema: [Cutouts catalog FITS file](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/merdpd/dpcards/mer_finalcatalog.html#cutouts-catalog-fits-file)
- Example column name: `CORNER_0_RA` --> `cutouts_corner_0_ra`

```{code-cell}
mer_prefixes = ["mer_", "morph_", "cutouts_"]
mer_col_counts = {p: len([n for n in schema.names if n.startswith(p)]) for p in mer_prefixes}

print(f"MER tables: {sum(mer_col_counts.values())} columns total")
for prefix, count in mer_col_counts.items():
    print(f"  {prefix}: {count}")
```

### 3.3 PHZ tables

The Euclid PHZ processing function produced eight tables.
The reference paper is [Euclid Collaboration: Tucci et al., 2025](https://arxiv.org/pdf/2503.15306) (hereafter, Tucci).
The tables, original schemas, and example column name transformations are as follows.

**Photo Z (phz)**
- Described in Tucci Sec. 5 ("phz_photo_z")
- Original schema: [Photo Z catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/phzdpd/dpcards/phz_phzpfoutputcatalog.html#photo-z-catalog)
- Example column name: `PHZ_PDF` --> `phz_phz_pdf`

**Classifications (class)**
- Described in Tucci Sec. 4 ("phz_classification")
- Original schema: [Classification catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/phzdpd/dpcards/phz_phzpfoutputforl3.html#classification-catalog)
- Example column name: `PHZ_CLASSIFICATION` --> `class_phz_classification`

**Galaxy Physical Parameters (physparam)**
- Described in Tucci Sec. 6 (6.1; "phz_physical_parameters")
- Original schema: [Physical Parameters catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/phzdpd/dpcards/phz_phzpfoutputforl3.html#physical-parameters-catalog)
- Example column name: `PHZ_PP_MEDIAN_REDSHIFT` --> `physparam_phz_pp_median_redshift`

**Galaxy SEDs (galaxysed)**
- Described in Tucci App. B (B.1 "phz_galaxy_sed")
- Original schema: [Galaxy SED catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/phzdpd/dpcards/phz_phzpfoutputcatalog.html#galaxy-sed-catalog)
- Example column name: `FLUX_4900_5000` --> `galaxysed_flux_4900_5000`

**QSO Physical Parameters (physparamqso)**
- Described in Tucci Sec. 6 (6.2; "phz_qso_physical_parameters")
- Original schema: [QSO Physical Parameters catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/phzdpd/dpcards/phz_phzpfoutputforl3.html#qso-physical-parameters-catalog)
- Example column name: `E(B-V)` --> `physparamqso_e_b_v_`

**Star Parameters (starclass)**
- Described in Tucci Sec. 6 (6.3; "phz_star_template")
- Original schema: [Star Template](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/phzdpd/dpcards/phz_phzpfoutputforl3.html#star-template)
- Example column name: `FLUX_VIS_Total_Corrected` --> `starclass_flux_vis_total_corrected`

**Star SEDs (starsed)**
- Described in Tucci App. B (B.1 "phz_star_sed")
- Original schema: [Star SED catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/phzdpd/dpcards/phz_phzpfoutputcatalog.html#star-sed-catalog)
- Example column name: `FLUX_4900_5000` --> `starsed_flux_4900_5000`

**NIR Physical Parameters (physparamnir)**
- Described in Tucci Sec. 6 (6.4; "phz_nir_physical_parameters")
- Original schema: [NIR Physical Parameters catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/phzdpd/dpcards/phz_phzpfoutputforl3.html#nir-physical-parameters-catalog)
- Example column name: `E(B-V)` --> `physparamnir_e_b_v_`

```{code-cell}
phz_prefixes = ["phz_", "class_", "physparam_", "galaxysed_", "physparamqso_", "starclass_", "starsed_", "physparamnir_"]
phz_col_counts = {p: len([n for n in schema.names if n.startswith(p)]) for p in phz_prefixes}

print(f"PHZ tables: {sum(phz_col_counts.values())} columns total")
for prefix, count in phz_col_counts.items():
    print(f"  {prefix}: {count}")
```

### 3.4 SPE tables

The Euclid SPE processing function produced three tables that are included in the HATS Catalog.
The reference paper is [Euclid Collaboration: Le Brun et al., 2025](https://arxiv.org/pdf/2503.15308) (hereafter, Le Brun).
The tables, original schemas, and example column name transformations are as follows.

**Spectroscopic Redshifts (z)**
- Described in Le Brun Sec. 2 ("spectro_zcatalog_spe_quality", "spectro_zcatalog_spe_classification", "spectro_zcatalog_spe_galaxy_candidates", "spectro_zcatalog_spe_star_candidates", and "spectro_zcatalog_spe_qso_candidates")
- Original schema: [Redshift catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/spedpd/dpcards/spe_spepfoutputcatalog.html#redshift-catalog)
- Example column name: `SPE_PDF` --> `z_galaxy_candidates_spe_pdf_rank0`

```{important}
This table required special handling.
The original FITS files contain spectroscopic redshift estimates for GALAXY_CANDIDATES, STAR_CANDIDATES, and QSO_CANDIDATES (HDUs 3, 4, and 5 respectively) with up to 5 estimates per 'object_id', per HDU.
These were pivoted so that there is one row per 'object_id' to facilitate table joins.
The resulting columns were named by combining the table name (z), the HDU name, the original column name, and the rank of the redshift estimate (i.e., the value in the original 'SPE_RANK' column).
```

**Spectral Line Measurements (lines)**
- Described in Le Brun Sec. 5 ("spectro_line_features_catalog_spe_line_features_cat"). _Notice that lines were identified assuming **galaxy** regardless of the classification._
- Original schema: [Lines catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/spedpd/dpcards/spe_spepfoutputcatalog.html#lines-catalog)
- Example column name: `SPE_LINE_FLUX_GF` --> `lines_spe_line_flux_gf_rank0_halpha`

```{important}
The HATS Catalog only includes rows from HDU1 with SPE_LINE_NAME == Halpha.
Similar to the 'z' table, there are up to 5 sets of columns per 'object_id', one set per redshift estimate.
Column names have been appended with both the rank and the line name.
```

**Models (models)**
- Described in Le Brun Sec. 5 ("spectro_model_catalog_spe_lines_catalog")
- Original schema: [Models catalog](http://st-dm.pages.euclid-sgs.uk/data-product-doc/dmq1/spedpd/dpcards/spe_spepfoutputcatalog.html#models-catalog)
- Example column name: `SPE_VEL_DISP_E` --> `models_galaxy_spe_vel_disp_e_rank0`

```{important}
The HATS Catalog only includes HDU2 -- the model parameters for galaxy solutions.
This table has the same structure as 'z'.
In addition to the table name, 'galaxy' has been prepended to the column names.
```

```{code-cell}
spe_prefixes = ["z_", "lines_", "models_"]
spe_col_counts = {p: len([n for n in schema.names if n.startswith(p)]) for p in spe_prefixes}

print(f"SPE tables: {sum(spe_col_counts.values())} columns total")
for prefix, count in spe_col_counts.items():
    print(f"  {prefix[:-1]}: {count}")
```

### 3.5 Additional columns

Several columns have been added to this dataset that are not in the original Euclid Q1 tables.

Euclid columns:

- `tileid` : ID of the Euclid Tile the object was detected in.

HEALPix columns:

These are HEALPix pixel indexes corresponding to the object's RA and Dec coordinates.
They are useful for spatial queries.

- `_healpix_9` : HEALPix order 9 pixel index.
- `_healpix_19` : HEALPix order 19 pixel index.
- `_healpix_29` : (hats column) HEALPix order 29 pixel index.

HATS columns:

These are used for the HATS partitioning.

- `Norder` : (hats column) HEALPix order at which the data is partitioned.
- `Npix` : (hats column) HEALPix pixel index at order Norder.
- `Dir` : (hats column) Integer equal to 10_000 * floor[Npix / 10_000].

+++

### 3.6 Find columns of interest

The HEALPix, object ID, and tile ID columns appear first:

```{code-cell}
schema.names[:5]
```

The HATS partitioning columns appear at the end of the schema.
They are also accessible from the PyArrow dataset partitioning object.

```{code-cell}
schema.names[-3:]
```

To find all columns from a given table, search for column names that start with the table name followed by an underscore.

```{code-cell}
# Find all column names from the phz table.
phz_columns = [name for name in schema.names if name.startswith("phz_")]

print(f"{len(phz_columns)} columns from the PHZ table. First four are:")
phz_columns[:4]
```

Column metadata includes unit and description.

```{code-cell}
schema.field("mer_flux_y_2fwhm_aper").metadata
```

Euclid Q1 offers many flux measurements, both from Euclid detections and from external ground-based surveys.
They are given in microjanskys, so all flux columns can be found by searching the metadata for this unit.

```{code-cell}
# Find all flux columns.
flux_columns = [field.name for field in schema if field.metadata[b"unit"] == b"uJy"]

print(f"{len(flux_columns)} flux columns. First four are:")
flux_columns[:4]
```

Columns associated with external surveys are identified by the inclusion of "ext" in the name.

```{code-cell}
external_flux_columns = [name for name in flux_columns if "ext" in name]
print(f"{len(external_flux_columns)} flux columns from external surveys. First four are:")
external_flux_columns[:4]
```

## About this notebook

**Authors:** Troy Raen (Developer; Caltech/IPAC-IRSA) and the IRSA Data Science Team.

**Updated:** 2025-11-25

**Contact:** [IRSA Helpdesk](https://irsa.ipac.caltech.edu/docs/help_desk.html) with questions or problems.
