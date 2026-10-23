# Sb-halide dimensionality prediction

Research code for binary prediction of inorganic Sb-halide connectivity:
`0 = 0D` and `1 = non-0D` (1D, 2D, or 3D).

## Project overview

This project studies the structural dimensionality of organic–inorganic
antimony-halide compounds. We first identify relevant compounds in chemistry
papers and extract their compositions, organic cations, structural information,
and reported dimensionalities. These literature data form the labeled dataset
used throughout the project.

We train Random Forest and support vector machine (SVM) classifiers to predict
whether inorganic Sb-halide connectivity is zero-dimensional or extended. We
also evaluate GPT models on the same held-out task using zero-shot, few-shot,
and all-shot prompting.

To understand the predictions, we calculate permutation and SHAP importance for
the ML models and ask the GPT models to rank a set of chemical and compositional
features to understand feature importance for each LLM. Finally, we apply the
best-performing ML and GPT configurations to ten
newly synthesized Sb-halide compounds.

## Repository layout

The directories follow the research workflow from literature extraction to
model evaluation and prediction:

- `data_extraction/`: PDF-to-dataset pipeline with evidence-linked Excel and JSON export.
- `data/`: prepared modeling datasets used by the downstream workflows.
- `ml_training/`: feature engineering, leakage-controlled splitting, Random Forest, and SVM training.
- `feature_importance/`: permutation and SHAP importance for the selected ML model.
- `llm_prediction/`: held-out LLM evaluation and in-context prediction for new compounds.
- `llm_feature_importance/`: paired feature-ranking experiments with and without labeled examples.
- `model_prediction/`: feature generation and full-data ML prediction for new compounds.
- `evaluation/`: ML/LLM comparisons and confusion-matrix plots.
- `common/`: shared plotting utilities.
- `results_ml/`: generated ML metrics, predictions, and fitted models.
- `results_llm/`: LLM evaluation results behind Figure 3.
- `figures/`: publication-figure scripts, generated publication figures, and diagnostic figures.

## Installation

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

SMI-TED must be available when creating molecular embeddings. Supply its
`smi_ted_light` directory with `--smi-ted-dir`; the preparation script can
download the required files from Hugging Face if the argument is omitted.

Set `OPENAI_API_KEY` before live data-extraction or LLM-prediction runs.

## Literature data extraction

The `data_extraction/` folder contains the code that builds the literature-derived
dataset used by the prediction workflows. It combines deterministic PDF parsing, BM25 retrieval,
validation, source citation tracking, and final export with schema-constrained LLM
stages for scientific interpretation.

```text
PDFs -> parse -> screen -> compound registry -> retrieve -> dossiers
     -> structured extraction -> verification -> final Excel + JSON datasets
```

Each extracted compound retains its supporting evidence text and page-level
source IDs. The pipeline produces one record per compound, including reported
formula and cation information, Sb-halide connectivity, dimensionality and
reasoning, synthesis evidence, verification results, and automatic flags.

The output should be treated as raw, machine-extracted data rather than a fully
curated dataset. Users should review and correct the extracted fields and
supporting evidence against the original papers before using the data for
analysis.

To run the extraction from the repository root:

1. Enter the extraction directory and create the local environment file:

   ```bash
   cd data_extraction
   cp .env.example .env
   ```

2. Open `data_extraction/.env` and replace the placeholder with your OpenAI API
   key:

   ```dotenv
   OPENAI_API_KEY="sk-..."
   ```

3. Create the input directory and copy the article PDFs into it:

   ```bash
   mkdir -p data/papers
   cp /path/to/your/pdfs/*.pdf data/papers/
   ```

   PDF filenames must contain their paper IDs, for example `P0001.pdf`. Nested
   directories under `data/papers/` are also supported.

4. Run the complete pipeline for selected papers:

   ```bash
   python -m src.run_pipeline all --paper-ids P0001,P0002
   ```

To process every PDF under `data/papers/`, run:

```bash
python -m src.run_pipeline all
```

The final stage writes both formats directly:

```text
data/output/final_dataset.xlsx
data/output/final_dataset.json
```

Intermediate JSONL artifacts and cached LLM responses are also created under
`data/`. Pipeline settings, model assignments, retrieval limits, and input/output
paths are defined in `config/pipeline.yaml`.

## ML workflow

The train/test split groups compounds by both source paper and organic cation, so no paper and no
cation structure appears in both sets. A few chemical compositions do occur in both sets, but they
correspond to different organic cation structures, so they are distinct compounds rather than duplicates.

```bash
python -m ml_training.prepare_data \
  --input data/merged_compound_cif_cation.xlsx \
  --output data/prepared_data.xlsx

python -m ml_training.train_models \
  --data data/prepared_data.xlsx \
  --out results_ml

python -m feature_importance.permutation_importance \
  --data data/prepared_data.xlsx \
  --results results_ml/ml_results.csv

python -m feature_importance.shap_importance \
  --data data/prepared_data.xlsx \
  --results results_ml/ml_results.csv
```

For the project-specific mixed-halide candidates:

```bash
python -m model_prediction.featurize_new_compounds \
  --input data/new_mixed_halide_sb_compounds.xlsx \
  --smi-ted-dir /path/to/smi_ted_light \
  --output data/new_compound_features.csv

python -m model_prediction.retrain_and_predict \
  --data data/prepared_data.xlsx \
  --new-features data/new_compound_features.csv \
  --out results_ml/new_compounds
```

## LLM prediction

```bash
python -m llm_prediction.evaluate_llms \
  --data data/prepared_data.xlsx \
  --out results_llm

python -m llm_prediction.predict_new_compounds \
  --data data/prepared_data.xlsx \
  --new-features data/new_compound_features.csv \
  --output results_llm/new_compound_predictions.csv
```

## LLM feature importance

The final experiment uses seven fixed feature categories, four LLMs, 30
controlled presentation orders, and paired conditions with and without the 321
training examples. The "with examples" condition below is also one of the two
panels in [Figure 4](#figure-4-feature-importance-for-the-best-performing-ml-and-llm-configurations);
the "no examples" condition and the `compare` step feed
[Figure 5](#figure-5-llm-feature-importance-without-training-examples-and-the-withwithout-data-difference).

```bash
python -m unittest -v llm_feature_importance.tests

python -m llm_feature_importance.collect_paired \
  --data data/prepared_data.xlsx

python -m llm_feature_importance.analyze --out llm_feature_importance/results
python -m llm_feature_importance.analyze --out llm_feature_importance/results_with_data
python -m llm_feature_importance.plot --out llm_feature_importance/results
python -m llm_feature_importance.plot --out llm_feature_importance/results_with_data
python -m llm_feature_importance.compare \
  --no-data llm_feature_importance/results \
  --with-data llm_feature_importance/results_with_data \
  --out llm_feature_importance/comparison
```

## Reproducing manuscript figures

### Figure 2: dataset composition and organic-cation chemical space

Figure 2 contains:

- **(a)** the 0D versus non-0D dimensionality distribution;
- **(b)** the dimensionality distribution within the F, Cl, Br, and I families; and
- **(c)** the RDKit-descriptor UMAP of the original and newly proposed organic cations.

Run from the repository root, after completing [Installation](#installation).
`figures/figrue_2/figure_2_reference.png` shows the expected result, so you can
compare it against your own output.

#### 1. Regenerate the UMAP coordinates for panel (c) (optional)

Panel (c)'s input coordinates are already in
`figures/figrue_2/reference_coordinates/`, so you can skip this step and go
straight to step 2. Run it if you want to
regenerate those coordinates yourself — to reproduce them from scratch, or to
add new proposed cations. `requirements.txt` pins the UMAP-relevant package
versions (`numpy`, `scipy`, `umap-learn`, `matplotlib`), so installing it as
specified reproduces `reference_coordinates/` exactly.

```bash
python figures/figrue_2/generate_umap_coordinates.py
```

This reads `data/merged_compound_cif_cation.xlsx` and
`figures/figrue_2/new_cations.csv`, then writes `umap_original_cations.csv`
and `umap_new_cations.csv` to `figures/figrue_2/generated_coordinates/`.

#### 2. Generate Figure 2

```bash
python figures/figrue_2/make_figure_2.py
```

This plots from `figures/figrue_2/reference_coordinates/` by default. If you
regenerated coordinates in step 1 and want to plot from those instead, pass
them explicitly:

```bash
python figures/figrue_2/make_figure_2.py \
  --coordinates-dir figures/figrue_2/generated_coordinates
```

Both the combined figure (`figure_2.png`/`.pdf`) and the individual panels are
written to `figures/figrue_2/output/`.

### Figure 3: LLM performance across prompting conditions and cation representations

Figure 3 shows test macro-F1 for five LLMs under zero-shot, 12-shot, and
all-shot prompting, using either IUPAC cation names or SMILES as input, against
a dashed best-ML reference line. `figures/figure_3/figure_3_reference.png`
shows the expected result, so you can compare it against your own output.

Figure 3 is built from LLM evaluation results, which are expensive to
regenerate (live API calls, billed per token) and not guaranteed to reproduce
exactly on a rerun (see caveats below), so `results_llm/llm_results.csv` is
committed as the reference data behind the published figure.

To generate Figure 3 from that reference data:

```bash
python -m evaluation.plot_llm_results
```

This writes several bar-chart variants to `figures/`; the one that is Figure 3
is `figures/llm_results_by_representation_final.png`. The others are
alternative/diagnostic views of the same results, not used in the manuscript.

#### Regenerating the LLM evaluation results (optional)

Only do this if you want to re-run the LLM evaluation yourself rather than use
the committed reference results. It requires an OpenAI API key with access to
the five evaluated models:

```bash
export OPENAI_API_KEY="sk-..."
python -m llm_prediction.evaluate_llms \
  --data data/prepared_data.xlsx \
  --out results_llm
```

Then point `plot_llm_results` at the new output:

```bash
python -m evaluation.plot_llm_results --llm-results results_llm/llm_results.csv
```

A rerun is not guaranteed to reproduce the published numbers exactly, for
reasons outside this code's control:

- Three of the five models are pinned to a dated snapshot
  (`gpt-4.1-2025-04-14`, `gpt-5.1-2025-11-13`, `gpt-5.5-2026-04-23`); the other
  two (`gpt-5.6-sol`, `gpt-6-astra`) are undated aliases, so OpenAI may change
  which underlying model they resolve to over time.
- Except for `gpt-4.1`, these are
  reasoning models, whose outputs are not guaranteed to be bit-for-bit
  reproducible across calls even on an unchanged snapshot with identical
  prompts.

### Figure 4: feature importance for the best-performing ML and LLM configurations

Figure 4 contains:

- **(a)** permutation importance for the best ML model (inorganic features plus
  SMI-TED embeddings with PCA-80); and
- **(b)** feature importance for each LLM's best setting, summarized by Borda
  score, from the "with examples" condition of the
  [LLM feature importance](#llm-feature-importance) experiment above.

`figures/figure_4/figure_4_reference.png` shows the expected result.

Panel (a) is produced by the `feature_importance.permutation_importance` step
in [ML workflow](#ml-workflow) above — run that first if you haven't
already. It's fully deterministic (fixed random seeds, pinned `scikit-learn`),
so no reference data is needed; just rerun it.

Panel (b) needs `llm_feature_importance.plot --out
llm_feature_importance/results_with_data`, from
[LLM feature importance](#llm-feature-importance) above. That command's input
(`model_feature_importance.csv`, `model_stability_kendall.csv`,
`intermodel_kendall.csv`) is committed as reference data in
`llm_feature_importance/results_with_data/`, since it comes from live,
costly LLM calls (the same reproducibility caveats as Figure 3 apply if you
regenerate it yourself via `collect_paired` and `analyze`).
`rankings_long.csv` is also committed there — the raw per-trial LLM output
(one row per model/trial/feature, with its rank and Borda score) that the
other three files are aggregated from.

Once both panels exist at their default paths
(`figures/ml_permutation_importance_no_title.png` and
`llm_feature_importance/results_with_data/figures/figure1_feature_importance_heatmap.png`),
compose them into Figure 4:

```bash
python figures/figure_4/make_figure_4.py
```

The combined figure (`figure_4.png`/`.pdf`) is written to
`figures/figure_4/output/`.

### Figure 5: LLM feature importance without training examples, and the with/without-data difference

Figure 5 contains:

- **(a)** LLM-assessed feature importance without training examples, from the
  "no examples" condition of the
  [LLM feature importance](#llm-feature-importance) experiment above; and
- **(b)** the change in feature importance between the with-data and
  without-data settings, from that experiment's `compare` step.

`figures/figure_5/figure_5_reference.png` shows the expected result.

Panel (a) needs `llm_feature_importance.plot --out
llm_feature_importance/results`, and panel (b) needs `llm_feature_importance.compare`
(both from [LLM feature importance](#llm-feature-importance) above). Both
read from CSVs — `model_feature_importance.csv`, `model_stability_kendall.csv`,
`intermodel_kendall.csv`, `rankings_long.csv`, and `run_manifest.json` — committed
as reference data in `llm_feature_importance/results/` (this figure's no-data
condition) and `llm_feature_importance/results_with_data/` (already used by
Figure 4), for the same reason as Figure 3 and 4's reference data: they come
from live, costly LLM calls. Unlike collection, `compare` itself is a
deterministic computation over those committed CSVs — no API calls — so it
always reproduces exactly from them.

Once both panels exist at their default paths
(`llm_feature_importance/results/figures/figure1_feature_importance_heatmap.png`
and `llm_feature_importance/comparison/comparison_delta_heatmap.png`), compose
them into Figure 5:

```bash
python figures/figure_5/make_figure_5.py
```

The combined figure (`figure_5.png`/`.pdf`) is written to
`figures/figure_5/output/`.
"""Reproduce Figure 2: dataset composition and organic-cation chemical space.

Panels
------
(a) Counts of 0D and non-0D compounds, with non-0D split into 1D/2D/3D.
(b) Dimensionality composition of the F, Cl, Br, and I subsets.
(c) UMAP of RDKit descriptors for original and newly proposed cations.

The script reads the curated compound workbook for panels (a) and (b), and the UMAP
coordinate CSVs created by generate_umap_coordinates.py for panel (c). It does not
require the large model-training feature table.
"""

from __future__ import annotations

import argparse
from pathlib import Path

import matplotlib

matplotlib.use("Agg")
import matplotlib.image as mpimg
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd


DIMS = ["0D", "1D", "2D", "3D"]
PURE_HALIDES = ["F", "Cl", "Br", "I"]
DIM_COLOR = {"0D": "#E69F00", "1D": "#0072B2", "2D": "#009E73", "3D": "#D55E00"}
INK, INK_SOFT, GRID, AXIS = "#0b0b0b", "#52514e", "#e3e2dc", "#c7c6bf"
VISUAL_FLOOR = 0.009


def configure_style() -> None:
    plt.rcParams.update(
        {
            "font.family": "sans-serif",
            "font.sans-serif": ["Helvetica", "Arial", "DejaVu Sans"],
            "font.size": 19,
            "axes.linewidth": 0.8,
            "axes.labelsize": 20,
            "xtick.labelsize": 19,
            "ytick.labelsize": 19,
            "legend.fontsize": 18,
            "text.color": INK,
            "axes.labelcolor": INK,
            "xtick.color": INK_SOFT,
            "ytick.color": INK_SOFT,
        }
    )


def style_axes(ax, grid_axis: str | None = "y") -> None:
    ax.set_facecolor("white")
    if grid_axis:
        ax.grid(axis=grid_axis, color=GRID, lw=0.8, zorder=0)
    ax.set_axisbelow(True)
    for spine in ("top", "right"):
        ax.spines[spine].set_visible(False)
    for spine in ("left", "bottom"):
        ax.spines[spine].set_color(AXIS)


def declutter(centers: list[float], min_gap: float) -> list[float]:
    positions = list(centers)
    for index in range(1, len(positions)):
        if positions[index] - positions[index - 1] < min_gap:
            positions[index] = positions[index - 1] + min_gap
    for index in range(len(positions) - 2, -1, -1):
        if positions[index + 1] - positions[index] < min_gap:
            positions[index] = positions[index + 1] - min_gap
    return positions


def save_figure(fig: plt.Figure, output: Path) -> None:
    fig.savefig(output, dpi=600, facecolor="white", bbox_inches="tight", pad_inches=0.08)


def plot_panel_a(data: pd.DataFrame, output: Path) -> None:
    counts = data["dimensionality"].value_counts().reindex(DIMS, fill_value=0)
    total = int(counts.sum())
    nonzero_dims = ["1D", "2D", "3D"]
    nonzero_total = int(counts[nonzero_dims].sum())

    fig, ax = plt.subplots(figsize=(5.4, 6.0))
    fig.patch.set_facecolor("white")
    bar_width, x_0d, x_nonzero = 0.4, 0.0, 1.0
    y_max = max(int(counts["0D"]), nonzero_total) * 1.18

    ax.bar(x_0d, counts["0D"], width=bar_width, color=DIM_COLOR["0D"], zorder=3)

    true_heights = {dim: int(counts[dim]) for dim in nonzero_dims}
    visual_series = pd.Series({dim: float(counts[dim]) for dim in nonzero_dims})
    min_visual = y_max * VISUAL_FLOOR
    deficits = {dim: max(min_visual - value, 0.0) for dim, value in true_heights.items()}
    if any(deficits.values()):
        donor = max(true_heights, key=true_heights.get)
        adjusted = visual_series.copy()
        adjusted[donor] = max(adjusted[donor] - sum(deficits.values()), 0.0)
        for dim, deficit in deficits.items():
            adjusted[dim] = adjusted.get(dim, 0.0) + deficit
        visual_series = adjusted

    bottom = 0.0
    segments = []
    for dim in nonzero_dims:
        height = float(visual_series[dim])
        ax.bar(x_nonzero, height, width=bar_width, bottom=bottom, color=DIM_COLOR[dim], zorder=3)
        segments.append((dim, true_heights[dim], bottom + height / 2))
        bottom += height

    ax.set_ylim(0, y_max)
    ax.set_xlim(-0.55, x_nonzero + 0.75)
    ax.set_xticks([x_0d, x_nonzero], ["0D", "non-0D"])
    ax.set_ylabel("number of compounds")
    style_axes(ax)

    ax.text(
        x_0d,
        int(counts["0D"]) + y_max * 0.015,
        f"{int(counts['0D'])}\n{counts['0D'] / total * 100:.1f}%",
        ha="center",
        va="bottom",
        fontsize=19,
        linespacing=1.25,
    )

    right_edge = x_nonzero + bar_width / 2
    label_y = declutter([center for _, _, center in segments], y_max * 0.14)
    for (dim, value, center), y_value in zip(segments, label_y):
        ax.plot([right_edge, right_edge + 0.08], [center, y_value], color=INK_SOFT, lw=0.7, zorder=4)
        ax.text(
            right_edge + 0.11,
            y_value,
            f"{dim}: {value}\n{value / total * 100:.1f}%",
            ha="left",
            va="center",
            fontsize=18,
            linespacing=1.2,
        )

    fig.tight_layout()
    save_figure(fig, output)
    plt.close(fig)


def plot_panel_b(data: pd.DataFrame, output: Path) -> None:
    subset = data[data["Sb_halide"].isin(PURE_HALIDES)]
    counts = (
        subset.groupby(["Sb_halide", "dimensionality"], observed=False)
        .size()
        .unstack(fill_value=0)
        .reindex(index=PURE_HALIDES, columns=DIMS, fill_value=0)
    )
    totals = counts.sum(axis=1)
    fractions = counts.div(totals, axis=0)

    visual = fractions.copy()
    min_visual = 1.10 * VISUAL_FLOOR
    for halide in PURE_HALIDES:
        row = fractions.loc[halide]
        deficits = {dim: min_visual - row[dim] for dim in DIMS if 0 < row[dim] < min_visual}
        if deficits:
            visual.loc[halide, row.idxmax()] -= sum(deficits.values())
            for dim, deficit in deficits.items():
                visual.loc[halide, dim] += deficit

    x_values = np.arange(len(PURE_HALIDES))
    bottoms = np.zeros(len(PURE_HALIDES))
    fig, ax = plt.subplots(figsize=(5.4, 6.0))
    fig.patch.set_facecolor("white")
    for dim in DIMS:
        visible_values = visual[dim].to_numpy()
        true_values = fractions[dim].to_numpy()
        ax.bar(
            x_values,
            visible_values,
            0.52,
            bottom=bottoms,
            color=DIM_COLOR[dim],
            label=dim,
            zorder=3,
        )
        for index, (visible, true) in enumerate(zip(visible_values, true_values)):
            if true > 0.06:
                text_color = "white" if visible > 0.20 else "black"
                ax.text(
                    x_values[index],
                    bottoms[index] + visible / 2,
                    f"{true * 100:.0f}%",
                    ha="center",
                    va="center",
                    color=text_color,
                    fontsize=18,
                    fontweight="medium",
                )
        bottoms += visible_values

    for index, total in enumerate(totals.to_numpy()):
        ax.text(x_values[index], 1.02, f"n = {int(total)}", ha="center", va="bottom", fontsize=18, color=INK_SOFT)
    ax.set_ylim(0, 1.09)
    ax.set_xlim(-0.6, len(PURE_HALIDES) - 0.4)
    ax.set_xticks(x_values, PURE_HALIDES)
    ax.set_ylabel("percentage of compounds")
    ax.set_yticks([0, 0.25, 0.5, 0.75, 1.0], ["0", "25%", "50%", "75%", "100%"])
    ax.legend(
        loc="lower center",
        bbox_to_anchor=(0.5, 1.02),
        ncol=4,
        frameon=False,
        handlelength=1.2,
        handleheight=1.2,
        columnspacing=1.4,
    )
    style_axes(ax)
    fig.tight_layout()
    save_figure(fig, output)
    plt.close(fig)


def plot_panel_c(original: pd.DataFrame, proposed: pd.DataFrame, output: Path) -> None:
    fig, ax = plt.subplots(figsize=(5.4, 6.0))
    fig.patch.set_facecolor("white")
    style_axes(ax, grid_axis=None)
    ax.grid(True, color=GRID, lw=0.6, zorder=0)
    ax.scatter(
        original["UMAP_1"], original["UMAP_2"], s=22, alpha=0.9,
        label="Original cations", color="#D55E00", edgecolors="none", zorder=2,
    )
    new_only = proposed[~proposed["in_original"]]
    overlap = proposed[proposed["in_original"]]
    ax.scatter(
        new_only["UMAP_1"], new_only["UMAP_2"], s=44, marker="o",
        label="New cations", facecolors="none", edgecolors="#0072B2", linewidth=1.8, zorder=3,
    )
    ax.scatter(
        overlap["UMAP_1"], overlap["UMAP_2"], s=44, marker="s",
        label="Overlapping cations", facecolors="none", edgecolors="#009E73", linewidth=1.8, zorder=3,
    )
    ax.set_xlabel("UMAP 1")
    ax.set_ylabel("UMAP 2")
    ax.legend(frameon=True, facecolor="white", edgecolor="none", framealpha=0.85, loc="lower left")
    fig.tight_layout()
    save_figure(fig, output)
    plt.close(fig)


def compose_figure(panel_paths: list[Path], output_dir: Path) -> None:
    fig, axes = plt.subplots(1, 3, figsize=(16.2, 6.0))
    fig.patch.set_facecolor("white")
    fig.subplots_adjust(left=0.015, right=0.995, top=0.995, bottom=0.005, wspace=0.03)
    label_x = {"a": -0.045, "b": 0.01, "c": 0.01}
    for ax, label, path in zip(axes, ["a", "b", "c"], panel_paths):
        ax.imshow(mpimg.imread(path))
        ax.axis("off")
        ax.text(
            label_x[label], 0.99, f"({label})", transform=ax.transAxes,
            fontsize=24, fontweight="bold", ha="left", va="top", clip_on=False,
        )
    fig.savefig(output_dir / "figure_2.png", dpi=600, facecolor="white")
    fig.savefig(output_dir / "figure_2.pdf", facecolor="white")
    plt.close(fig)


def parse_args() -> argparse.Namespace:
    script_dir = Path(__file__).resolve().parent
    repository_root = script_dir.parents[1]
    workspace_root = script_dir.parents[2]

    def default_input(filename: str) -> Path:
        """Support both the public-repository layout and this analysis workspace."""
        candidates = [
            repository_root / filename,
            repository_root / "data" / filename,
            workspace_root / filename,
        ]
        return next((path for path in candidates if path.exists()), candidates[0])

    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument(
        "--data",
        type=Path,
        default=default_input("merged_compound_cif_cation.xlsx"),
        help="Curated compound workbook (default: data/merged_compound_cif_cation.xlsx).",
    )
    parser.add_argument(
        "--coordinates-dir",
        type=Path,
        default=script_dir / "reference_coordinates",
        help=(
            "Directory containing umap_original_cations.csv and "
            "umap_new_cations.csv (default: published reference_coordinates)."
        ),
    )
    parser.add_argument("--output", type=Path, default=script_dir / "output")
    return parser.parse_args()


def main() -> None:
    args = parse_args()
    configure_style()
    args.output.mkdir(parents=True, exist_ok=True)

    data = pd.read_excel(args.data)
    required = {"dimensionality", "Sb_halide"}
    missing = required - set(data.columns)
    if missing:
        raise ValueError(f"{args.data} is missing required columns: {sorted(missing)}")
    data["dimensionality"] = data["dimensionality"].astype(str).str.strip()
    data = data[data["dimensionality"].isin(DIMS)].copy()
    if len(data) != 400:
        print(f"Warning: expected 400 known-dimensionality compounds, found {len(data)}")

    panel_a = args.output / "panel_a_dimensionality_counts.png"
    panel_b = args.output / "panel_b_dimensionality_by_halide.png"
    panel_c = args.output / "panel_c_cation_umap.png"
    plot_panel_a(data, panel_a)
    plot_panel_b(data, panel_b)

    reference_original = args.coordinates_dir / "umap_original_cations.csv"
    reference_new = args.coordinates_dir / "umap_new_cations.csv"
    missing_coordinates = [path for path in (reference_original, reference_new) if not path.exists()]
    if missing_coordinates:
        raise FileNotFoundError(
            f"Missing UMAP coordinate file(s): {missing_coordinates}. "
            "Run figures/figrue_2/generate_umap_coordinates.py first."
        )

    original = pd.read_csv(reference_original)
    proposed = pd.read_csv(reference_new)
    print("Using the saved UMAP coordinate datasets for panel (c).")
    original[["canonical_unique_cation", "UMAP_1", "UMAP_2"]].to_csv(
        args.output / "umap_original_cations.csv", index=False
    )
    proposed[["cation_name", "SMILES", "in_original", "UMAP_1", "UMAP_2"]].to_csv(
        args.output / "umap_new_cations.csv", index=False
    )
    plot_panel_c(original, proposed, panel_c)
    compose_figure([panel_a, panel_b, panel_c], args.output)
    print(f"Saved Figure 2 and its component panels to {args.output.resolve()}")


if __name__ == "__main__":
    main()
