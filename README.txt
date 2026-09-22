================================================================================
CLAM PREPROCESSING GUIDE
From raw Whole Slide Images (WSI) to 224 x 224 patches
================================================================================

ATTRIBUTION
-----------
This repository bundles the preprocessing-only subset of the CLAM codebase
(https://github.com/mahmoodlab/CLAM), created by Mahmood Lab. All code files
here (create_patches_fp.py, extract_features_fp.py, build_preset.py,
wsi_core/, dataset_modules/dataset_h5.py, models/, utils/, presets/) are
copied unmodified from that project and remain licensed under GPLv3 (see
LICENSE.md). The full CLAM project also includes a downstream MIL training/
evaluation pipeline (main.py, eval.py, model_clam.py, etc.) which is
intentionally NOT included here, since this repo's scope is preprocessing
only. For the full framework, use the upstream repo directly.

REPO CONTENTS
-------------
    create_patches_fp.py         segment tissue + generate patch coordinates
    extract_features_fp.py       crop/resize patches + encode into features
    build_preset.py              helper to build a custom preset .csv
    presets/                     tuned segmentation parameter presets
    wsi_core/                    WSI reading, segmentation, patch coord logic
    dataset_modules/dataset_h5.py  patch-coordinate/image dataset classes
    models/                      image encoder loader (get_encoder)
    utils/                       shared helpers (file I/O, transforms, constants)
    env.yml                      conda environment definition
    LICENSE.md                   GPLv3 (from upstream CLAM)

This README documents the preprocessing pipeline in this repo:
raw WSI files (.svs / .tif / .ndpi / ...) -> tissue segmentation -> patch
coordinates -> (optional) extracted 224x224 patch images / feature vectors.

The two scripts involved are:
    create_patches_fp.py     -> segments tissue and records patch coordinates
    extract_features_fp.py   -> reads the coordinates back from the WSI and
                                 produces the actual 224x224 patch images,
                                 immediately encoding them into feature vectors

CLAM's patching step does NOT save patch images to disk by default -- it only
saves (x, y) pixel coordinates in an .h5 file (this keeps storage small,
since a WSI can contain 10,000+ patches). Patch images are generated
on-the-fly, at read time, either during feature extraction or with the small
helper snippet given in Section 5 if you want literal .png/.jpg 224x224 tiles
saved to disk.

--------------------------------------------------------------------------------
0. ENVIRONMENT SETUP
--------------------------------------------------------------------------------

conda env create -f env.yml
conda activate clam_latest

This installs openslide, torch, timm, h5py, opencv, etc. (see env.yml).
openslide also needs the system-level OpenSlide library; on Ubuntu:

    sudo apt-get install openslide-tools

--------------------------------------------------------------------------------
1. INPUT LAYOUT
--------------------------------------------------------------------------------

Put all raw slide files in a single flat folder, e.g.:

    DATA_DIRECTORY/
        slide_1.svs
        slide_2.svs
        ...

Supported formats are whatever OpenSlide supports (.svs, .tif, .ndpi, .mrxs,
.vms, .vmu, .scn, .qptiff, etc).

--------------------------------------------------------------------------------
2. STEP 1 -- SEGMENT TISSUE + GENERATE PATCH COORDINATES
--------------------------------------------------------------------------------

Run create_patches_fp.py with --patch_size 224 --step_size 224 so that the
coordinate grid itself is laid out in 224x224, non-overlapping steps at
patch_level 0 (full/native resolution):

    python create_patches_fp.py \
        --source DATA_DIRECTORY \
        --save_dir RESULTS_DIRECTORY \
        --patch_size 224 \
        --step_size 224 \
        --patch_level 0 \
        --seg --patch --stitch

Flags:
    --source        folder of raw WSIs (Section 1)
    --save_dir      output folder (created if missing)
    --patch_size    width/height in pixels of each patch AT patch_level
                    (224 => 224x224 patches)
    --step_size     stride between patch origins; equal to patch_size means
                    no overlap. Use a smaller value (e.g. 112) for 50% overlap
    --patch_level   WSI pyramid level to patch at (0 = highest/native
                    resolution -- almost always what you want for 224x224
                    tiles at full magnification)
    --seg           run tissue segmentation (Otsu/thresholding) first
    --patch         extract patch coordinates from the segmented tissue
    --stitch        build a downsampled "stitched" preview image per slide,
                    useful for sanity-checking coverage (not required)
    --preset        optional .csv from presets/ with tuned segmentation
                    parameters for a given tissue type, e.g.:
                        --preset tcga.csv
                    (see presets/tcga.csv, presets/bwh_biopsy.csv,
                    presets/bwh_resection.csv)
    --process_list  optional .csv to control per-slide params / which slides
                    to (re)process (auto-generated as process_list_autogen.csv
                    inside save_dir after every run, and reusable via this flag)

Output layout in RESULTS_DIRECTORY:

    RESULTS_DIRECTORY/
        masks/                     one .jpg per slide -- visualized tissue mask
        patches/                   one .h5 per slide -- patch (x, y) coords only
        stitches/                  one .jpg per slide -- downsampled patch-grid preview
        process_list_autogen.csv   per-slide segmentation/patch parameters + status

Each patches/<slide_id>.h5 file contains a dataset "coords" of shape (N, 2)
(pixel coordinates at patch_level) plus attributes recording patch_size,
patch_level, and the slide's downsample factors -- this is what
extract_features_fp.py (or the standalone snippet in Section 5) reads to
crop out the actual 224x224 images later.

Tips:
    - Inspect masks/ and stitches/ after a first run on a couple of slides to
      confirm segmentation is picking up tissue correctly before processing
      the whole cohort. Bad masks usually mean you need a different --preset
      or seg_level.
    - --no_auto_skip forces reprocessing of slides that already have a
      matching .h5 in patches/ (by default those are skipped).

--------------------------------------------------------------------------------
3. STEP 2 -- BUILD THE SLIDE LIST CSV
--------------------------------------------------------------------------------

extract_features_fp.py needs a CSV with a "slide_id" column (no extension),
listing every slide to process. You can build one straight from the raw
source folder:

    python - <<'PY'
import os, pandas as pd
source = "DATA_DIRECTORY"
slide_ext = ".svs"
slide_ids = [os.path.splitext(f)[0] for f in sorted(os.listdir(source))
             if f.endswith(slide_ext)]
pd.DataFrame({"slide_id": slide_ids}).to_csv("dataset_csv/my_slides.csv", index=False)
PY

--------------------------------------------------------------------------------
4. STEP 3 -- EXTRACT 224x224 PATCHES + FEATURES
--------------------------------------------------------------------------------

    python extract_features_fp.py \
        --data_h5_dir RESULTS_DIRECTORY \
        --data_slide_dir DATA_DIRECTORY \
        --csv_path dataset_csv/my_slides.csv \
        --feat_dir FEATURES_DIRECTORY \
        --slide_ext .svs \
        --model_name resnet50_trunc \
        --target_patch_size 224 \
        --batch_size 256

Flags:
    --data_h5_dir       RESULTS_DIRECTORY from Step 1 (it looks for
                        <data_h5_dir>/patches/<slide_id>.h5)
    --data_slide_dir    same DATA_DIRECTORY as Step 1 (raw WSIs)
    --csv_path          the slide list CSV from Section 3
    --feat_dir          where extracted features are written
    --slide_ext         file extension of your raw slides (.svs, .tif, ...)
    --model_name        image encoder used to embed each patch:
                        resnet50_trunc | uni_v1 | conch_v1
    --target_patch_size resize each cropped patch to this size (in pixels)
                        before feeding the encoder -- set to 224 to match
                        224x224 patches. If Step 1 already used
                        --patch_size 224, this is just a no-op resize
                        confirming the size; if Step 1 used a larger
                        patch_size (e.g. 256, common for TCGA-style
                        pipelines), this is what actually downsizes each
                        crop to 224x224 before encoding.
    --batch_size        patches per encoder forward pass (GPU memory permitting)

Output layout in FEATURES_DIRECTORY:

    FEATURES_DIRECTORY/
        h5_files/<slide_id>.h5   per-patch 'features' + 'coords' datasets
        pt_files/<slide_id>.pt   torch tensor of stacked patch features
                                 (this is what main.py / Generic_MIL_Dataset
                                 expects at data_root_dir/<task>_resnet_features)

This is the standard CLAM MIL pipeline: it never writes plain 224x224 image
files to disk, it goes straight from coordinates -> cropped+resized tensor ->
encoder -> feature vector, to keep storage down (a WSI with thousands of
patches would otherwise generate gigabytes of small images).

--------------------------------------------------------------------------------
5. (OPTIONAL) SAVING ACTUAL 224x224 IMAGE FILES TO DISK
--------------------------------------------------------------------------------

If you specifically need literal 224x224 .png/.jpg patch images (e.g. for
training a plain CNN classifier instead of the MIL pipeline, or for manual
inspection), read the coordinates back yourself with Whole_Slide_Bag_FP and
dump each tile:

    import os, h5py, openslide
    from PIL import Image

    slide_id   = "slide_1"
    wsi_path   = f"DATA_DIRECTORY/{slide_id}.svs"
    h5_path    = f"RESULTS_DIRECTORY/patches/{slide_id}.h5"
    out_dir    = f"PATCH_IMAGES_DIRECTORY/{slide_id}"
    patch_size = 224   # must match --patch_size used in Step 1

    os.makedirs(out_dir, exist_ok=True)
    wsi = openslide.open_slide(wsi_path)

    with h5py.File(h5_path, "r") as f:
        coords = f["coords"][:]
        patch_level = f["coords"].attrs["patch_level"]

    for x, y in coords:
        img = wsi.read_region((int(x), int(y)), patch_level,
                               (patch_size, patch_size)).convert("RGB")
        img.save(os.path.join(out_dir, f"{x}_{y}.png"))

If --patch_size in Step 1 was not already 224, resize before saving:

    img = img.resize((224, 224), Image.BICUBIC)

Note: doing this for a full cohort can produce very large numbers of files
and a lot of disk usage -- prefer the h5/pt feature pipeline (Section 4)
unless you have a specific reason to keep raw image tiles.

--------------------------------------------------------------------------------
6. QUICK REFERENCE -- MINIMUM COMMANDS FOR 224x224 PATCHES END-TO-END
--------------------------------------------------------------------------------

    conda activate clam_latest

    python create_patches_fp.py \
        --source DATA_DIRECTORY \
        --save_dir RESULTS_DIRECTORY \
        --patch_size 224 --step_size 224 --patch_level 0 \
        --seg --patch --stitch

    # build dataset_csv/my_slides.csv (Section 3), then:

    python extract_features_fp.py \
        --data_h5_dir RESULTS_DIRECTORY \
        --data_slide_dir DATA_DIRECTORY \
        --csv_path dataset_csv/my_slides.csv \
        --feat_dir FEATURES_DIRECTORY \
        --slide_ext .svs \
        --model_name resnet50_trunc \
        --target_patch_size 224

The resulting FEATURES_DIRECTORY/pt_files/*.pt files are ready to be used as
data_root_dir input to main.py for CLAM/MIL training (see main.py's
--data_root_dir and --task arguments).

--------------------------------------------------------------------------------
7. TROUBLESHOOTING
--------------------------------------------------------------------------------

- Empty / mostly-background masks in masks/: try a different --preset
  (presets/tcga.csv, bwh_biopsy.csv, bwh_resection.csv) or tune seg_level /
  sthresh / a_t via a custom process_list csv.
- "level_dim likely too large for successful segmentation": the auto-picked
  seg_level is too high resolution; pass a process_list csv with an explicit
  lower-resolution seg_level for that slide.
- Slide skipped with "already exist in destination location": Step 1 skips
  slides that already have a matching patches/<slide_id>.h5; pass
  --no_auto_skip to force reprocessing.
- Feature extraction skips a slide: extract_features_fp.py skips slides that
  already have a pt_files/<slide_id>.pt; pass --no_auto_skip to force
  reprocessing.
================================================================================
