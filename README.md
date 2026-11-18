<p align="center">
  <img src="docs/logo.png" alt="Ep2Format" width="320">
</p>

Recommend **single-domain (VHH/nanobody)** or **conventional (Fab/IgG)** antibody
format for a selected epitope on an antigen structure.

Provide an antigen chain and residue positions. Ep2Format measures the patch's
geometry, optionally adds frozen ESM-2 sequence features, and reports a calibrated
format score alongside an independent depth/exposure lookup table.

This package contains the inference application and its two trained classifiers.
It does not include the research dataset or model-training pipeline.


## The format landscape

Epitope depth and solvent exposure help describe the surfaces associated with
single-domain and conventional binders in the source dataset.

![Antibody-format preference across epitope depth and solvent exposure](docs/surface-light.svg#gh-light-mode-only)
![Antibody-format preference across epitope depth and solvent exposure](docs/surface-dark.svg#gh-dark-mode-only)

## How the model works

Start with an antigen structure, a chain, and the residues of the epitope you
want to target. Ep2Format describes that patch using two sources of information:
sequence features from ESM-2 and geometry measured from the structure. A trained
classifier combines them to estimate the probability of a single-domain format.

![Frozen ESM-2 encoder and antibody-format classifier architecture](docs/architecture-light.svg#gh-light-mode-only)
![Frozen ESM-2 encoder and antibody-format classifier architecture](docs/architecture-dark.svg#gh-dark-mode-only)

### Sequence features from ESM-2

The antigen chain's sequence passes through a frozen ESM-2 encoder ([Lin et al., 2023](https://doi.org/10.1126/science.ade2574)), producing
1,280 values per residue at layer 12. The selected epitope positions are pooled
into two vectors: the epitope mean minus the chain mean, and the element-wise
maximum across epitope residues. Concatenating them gives **2,560 sequence
features**, including for patches whose residues are separated in sequence.

ESM's weights are not updated during classifier training. The current code
executes all 33 encoder layers and then selects layer 12; the diagram's removal
of later layers is a potential optimization, not an implemented one.

### Geometry descriptors

The structure provides measurements of the selected patch's shape, exposure, and
secondary structure. The bundled models reserve 20 feature slots for this block,
listed below. Cα denotes a residue's alpha-carbon position; SASA is
solvent-accessible surface area.

| Descriptor | Meaning | Units |
|---|---|---|
| `n_epitope_residues` | Number of resolved residues in the selected antigen patch | Count |
| `n_paratope_residues`* | Antibody residues within 4.5 Å of the antigen | Count |
| `n_segments` | Number of contiguous sequence runs in the epitope | Count |
| `longest_segment` | Length of the longest contiguous run | Residues |
| `linearity` | Longest run divided by the number of mapped epitope positions | Ratio |
| `planarity_rmsd_A` | RMS distance of epitope Cα positions from their best-fit plane | Å |
| `radius_of_gyration_A` | RMS distance of epitope Cα positions from their centroid | Å |
| `protrusion` | Mean distance from the antigen centroid, normalized by antigen radius of gyration | Ratio |
| `curvature_signed` | Signed inverse fitted-sphere radius; negative indicates concavity | Å⁻¹ |
| `sphere_radius_A` | Radius of the sphere fitted to epitope Cα positions | Å |
| `concavity` | Rim-minus-centre displacement along the outward normal; positive indicates a bowl | Å |
| `shape_eccentricity` | Coordinate anisotropy: $(s_1-s_3)/s_1$ | Ratio |
| `shape_sphericity` | Coordinate isotropy: $3s_3/(s_1+s_2+s_3)$ | Ratio |
| `hull_depth_mean` | Mean depth of epitope Cα positions beneath the antigen convex hull | Å |
| `hull_depth_max` | Maximum depth beneath that convex hull | Å |
| `mean_rel_sasa_free` | Mean residue SASA normalized by residue-specific maximum accessible area | Ratio |
| `frac_buried_residues` | Fraction of epitope residues with relative SASA below 0.25 | Fraction |
| `frac_helix` | Fraction of epitope residues annotated as α-helix | Fraction |
| `frac_sheet` | Fraction annotated as β-sheet | Fraction |
| `frac_coil` | Fraction annotated as coil/loop | Fraction |

Here $s_1 \ge s_2 \ge s_3$ are singular values of the centred epitope coordinates.

SASA uses Biotite's Shrake–Rupley implementation with ProtOr radii and
Tien et al.'s theoretical maximum accessible areas. Secondary structure uses
Biotite's Cα-based P-SEA implementation. See [References](#references).

*`n_paratope_residues` requires an antibody and is supplied as a missing value
during antigen-only inference. It remains in the preserved models' training
feature lists; see the [model card](docs/model-card.md).

### From features to a format score

The full model combines the 2,560 sequence features with the 20 geometry slots,
giving **2,580 inputs** to a gradient-boosted classifier. An isotonic calibrator
maps its raw probability to the reported single-domain score. The conventional
format score is its complement, $1-p$.

With `--no-esm`, the application uses a separately trained geometry-only model.
Both variants also report the independent depth/exposure lookup, so readers can
compare the learned prediction with an interpretable empirical summary. Scores
near 0.5 indicate ambiguity; the confidence label reflects distance from 0.5.

### ESM layer comparison

The original research compared representations from several ESM layers using
linear and MLP classifier heads. This experiment provides context for selecting
layer 12; the application uses the gradient-boosted classifier described above.

![Original ESM-2 layer comparison with linear and MLP classifiers](docs/esm-layers-light.svg#gh-light-mode-only)
![Original ESM-2 layer comparison with linear and MLP classifiers](docs/esm-layers-dark.svg#gh-dark-mode-only)

For reference, let $x \in \mathbb{R}^{2560}$ be the pooled ESM feature vector
after median imputation and standardization. The two classifier heads estimate
the probability of the single-domain label ($y=1$):

**Linear head (logistic regression):**

$$
P(y=1 \mid x) = \sigma(w^\top x + b)
$$

**MLP head (two hidden layers, 256 and 64 units):**

$$
h_1 = \mathrm{ReLU}(W_1x + b_1), \qquad
h_2 = \mathrm{ReLU}(W_2h_1 + b_2), \qquad
P(y=1 \mid x) = \sigma(w_3^\top h_2 + b_3)
$$

Here $\sigma(z)=1/(1+e^{-z})$ and $\mathrm{ReLU}(z)=\max(0,z)$,
applied element-wise. The linear head learns a weighted sum of the input features;
the MLP learns nonlinear combinations through its hidden layers. Both learn only
the head's weights and biases; the ESM encoder remains frozen.

Performance numbers in these figures come from the original research runs, not
a new evaluation of Ep2Format. See the [model card](docs/model-card.md) for the
saved model reports and the training/inference limitation.

## Installation

Use Python 3.11 or later, preferably in a virtual environment. From this directory:

```bash
python -m pip install .
```

This installs the geometry-only runtime. To use ESM sequence features:

```bash
python -m pip install '.[esm]'
```

The two classifier files are bundled in the package. ESM-2 weights are separate:
the first ESM prediction downloads the checkpoint through Hugging Face, unless it
is already cached. ESM chooses CUDA, Apple MPS, or CPU when available.

## Recommend a format

Experimental structure, geometry-only model:

```bash
ep2format --pdb 3P0G --chain A --residues 267-278,325-330 --no-esm
```

AlphaFold model by UniProt accession, using geometry and ESM:

```bash
ep2format --uniprot P07550 --residues 267-278,325-330
```

Local BinaryCIF file and machine-readable output:

```bash
ep2format --file antigen.bcif --chain A --residues 40-55 --no-esm --json
```

`python -m ep2format` is equivalent to `ep2format`. Run `ep2format --help` for options.
Use author residue numbering for PDB entries; AlphaFold generally uses UniProt
numbering. The default chain is the first chain in sorted order, so specifying
`--chain` is recommended for multichain structures. At least three requested
residues must remain resolved after trimming.

Only BinaryCIF local files are supported; text mmCIF and legacy PDB files must be
converted beforehand. Remote PDB and AlphaFold retrieval require network access.

For structures detected as predicted, `--plddt-min 70` is the default trimming
threshold. Detection uses a B-factor heuristic and can misclassify experimental
structures; `--plddt-min 0` disables trimming. ESM inference uses the first 1,600
sequence residues; training was restricted to sequences of at most 1,200 residues.

## Read the result

- **Topography:** pocket, groove, cleft, flat, protrusion, or mixed.
- **Depth and exposure:** hull depth in angstroms and relative solvent accessibility.
- **Lookup:** empirical single-domain frequency, enrichment against its reference
  population, and cluster-bootstrap interval.
- **Model:** single-domain probability, format call, and a confidence band based
  on distance from 0.5. This band is not a statistical confidence interval.

Near-0.5 predictions and disagreements between the lookup and model should be
treated as ambiguous. The CLI still emits a binary call; it does not enforce
abstention. Diagnostics are printed to stderr when `--json` is selected.

These scores describe format associations among solved PDB complexes. They do
not predict affinity, expression, developability, immunogenicity, or experimental
success. Ep2Format does not design antibody sequences or select epitope positions.

## Worked examples

Seven saved epitope examples illustrate the source project's format
recommendations, using experimental structures and an AlphaFold model.
These are historical illustrations, not an independent validation set.
The [saved example results and notes](examples/README.md) are included under
`examples/`; the original chain and residue selections are not recorded in that data.

![Seven worked epitope examples from the original research](docs/worked-examples-light.svg#gh-light-mode-only)
![Seven worked epitope examples from the original research](docs/worked-examples-dark.svg#gh-dark-mode-only)

## Lookup without a model

The lookup module itself needs only Python's standard library:

```python
from ep2format.lookup import recommend

result = recommend(depth_A=8.0, rel_sasa=0.30)
print(result["recommend"], result["enrichment"], result["ci"])
```

The lookup's reference rate is 21.65%; the classifiers' training subset has a
22.55% single-domain rate. They are separate empirical summaries.

## Package contents

| Path | Purpose |
|---|---|
| `ep2format/cli.py`, `__main__.py` | Command-line application |
| `ep2format/structure_io.py` | BinaryCIF retrieval and parsing |
| `ep2format/describe.py`, `geometry.py`, `topography.py` | Selected epitope descriptors |
| `ep2format/embeddings.py` | Frozen ESM-2 loader |
| `ep2format/model.py` | Calibrated classifier inference and serialization compatibility |
| `ep2format/lookup.py` | Independent two-variable lookup table |
| `ep2format/config.py`, `__init__.py` | Runtime configuration and package version |
| `ep2format/models/` | Two classifiers, feature metadata, and original reports |
| `docs/*.svg` | Original README figures in light and dark themes |
| `docs/model-card.md` | Provenance, architecture, and known limitations |
| `docs/migration.md` | What was retained and excluded from the source project |
| `examples/` | Seven saved worked-example results and reproduction notes |
| `tests/` | Offline regression checks |

Run offline checks with `python -m unittest discover -s tests -v` after installing.
Publish this directory as the repository root; the original project remains a
separate research archive. No Git history or remote configuration is included.

The `epitopex/` directory is a two-file serialization compatibility shim for the
bundled models. New code should import `ep2format`.

## License

Ep2Format is released under the [MIT License](LICENSE).
Third-party dependencies and separately downloaded model weights retain their
respective licenses. Scientific citation guidance does not add restrictions
to the MIT license.

## References

These references credit the methods, software, and structure resources used by
Ep2Format.

| Component | Reference |
|---|---|
| Frozen ESM-2 representations | Lin et al. (2023). *Evolutionary-scale prediction of atomic-level protein structure with a language model*. Science. [DOI](https://doi.org/10.1126/science.ade2574). |
| Structure parsing and analysis | Kunzmann and Hamacher (2018). *Biotite: a unifying open source computational biology framework in Python*. BMC Bioinformatics. [DOI](https://doi.org/10.1186/s12859-018-2367-z). |
| Solvent-accessible surface area | Shrake and Rupley (1973). *Environment and exposure to solvent of protein atoms. Lysozyme and insulin*. Journal of Molecular Biology. [DOI](https://doi.org/10.1016/0022-2836(73)90011-9). |
| ProtOr atomic radii | Tsai et al. (1999). *The packing density in proteins: standard radii and volumes*. Journal of Molecular Biology. [DOI](https://doi.org/10.1006/jmbi.1999.2829). |
| Relative SASA normalization | Tien et al. (2013). *Maximum Allowed Solvent Accessibilites of Residues in Proteins*. PLOS ONE. [DOI](https://doi.org/10.1371/journal.pone.0080635). |
| Secondary structure | Labesse et al. (1997). *P-SEA: a new efficient assignment of secondary structure from Cα trace of proteins*. Bioinformatics. [DOI](https://doi.org/10.1093/bioinformatics/13.3.291). |
| Classifiers and calibration | Pedregosa et al. (2011). *Scikit-learn: Machine Learning in Python*. JMLR 12:2825–2830. [Paper](https://jmlr.org/papers/v12/pedregosa11a.html). |
| Experimental structure resource | Burley et al. (2025). *Updated resources for exploring experimentally-determined PDB structures and Computed Structure Models at the RCSB Protein Data Bank*. Nucleic Acids Research. [DOI](https://doi.org/10.1093/nar/gkae1091). |
| AlphaFold method, when using predicted structures | Jumper et al. (2021). *Highly accurate protein structure prediction with AlphaFold*. Nature. [DOI](https://doi.org/10.1038/s41586-021-03819-2). |
| AlphaFold structure resource, when used | Varadi et al. (2024). *AlphaFold Protein Structure Database in 2024: providing structure coverage for over 214 million protein sequences*. Nucleic Acids Research. [DOI](https://doi.org/10.1093/nar/gkad1011). |

The [Biotite SASA documentation](https://www.biotite-python.org/latest/apidoc/biotite.structure.sasa.html)
and [secondary-structure documentation](https://www.biotite-python.org/latest/apidoc/biotite.structure.annotate_sse.html)
describe the implementations used here. Biotite notes that its P-SEA assignments
can differ from the original P-SEA program.
