API reference
=============

The ``shorkie`` package is a thin helper layer over ``baskerville-yeast``. It exists
so that model loading, sequence preparation, and variant scoring have exactly one
implementation — the same one used by the examples, the CLI, and the paper's own
eQTL scorers.

``shorkie.config``
------------------

Path resolution. Every filesystem path in the repository goes through here.

.. code-block:: python

   from shorkie import config

   config.load(path=None)      # -> Config  (LRU-cached)
   config.get("dotted.key", default=None)
   config.path("dotted.key")   # -> pathlib.Path
   config.repo_root()          # -> pathlib.Path

Resolution order: ``$SHORKIE_CONFIG`` → ``config/paths.yaml`` →
``config/paths.example.yaml``. ``${token}`` interpolation resolves against
environment variables first, then other dotted keys.

``shorkie.models.ensemble``
---------------------------

The model API.

.. code-block:: python

   N_DNA = 4; N_SPECIES = 166; NUM_FEATURES = 170; SCEREVISIAE_COL = 114

**fetch_1hot** (fasta, chrom, start, end, seq_len)
   Fetch a window and one-hot encode it; left-pads with ``N`` for negative starts.

**make_input** (fasta, chrom, start, end, seq_len=16384, species_col=114, mask_pos=None)
   Build the ``(seq_len, 170)`` model input: DNA one-hot in channels 0–3, species
   identity in 4–169. ``mask_pos`` zeroes the DNA channels at one position — the
   LM-style masking used for masked-base prediction.

**load_ensemble** (model_dir, params_file, target_index, num_folds=8, fold_subdir="f{fold}c0")
   Load an N-fold ensemble from ``{model_dir}/train/{fold_subdir}/train/model_best.h5``,
   sliced to ``target_index`` and wrapped with reverse-complement averaging.

**ensemble_predict** (models, x) → ``(1, 1, bins, tracks)``
   Average the fold predictions. Aliased as ``predict``.

**logSED** (y_ref, y_alt, gene_slice) → float
   ``log2(Σ_alt + 1) − log2(Σ_ref + 1)`` over the gene-body bins, averaged across tracks.

**logSED_per_track** (y_ref, y_alt, gene_slice) → ``(tracks,)``
   The same, per track — use this to ask which assays a variant affects.

``shorkie.helpers.yeast_helpers``
---------------------------------

Sequence, attribution, and plotting utilities used by the figure notebooks.

* ``make_seq_1hot`` — plain 4-channel one-hot (no species channel)
* ``process_sequence`` — build model input plus a gene-annotation slice
* ``predict_tracks`` — like ``ensemble_predict``, but concatenates folds instead of averaging
* ``get_ism``, ``get_ism_shuffle`` — in-silico mutagenesis
* ``get_prediction_gradient`` (and ``_w_rc`` / ``_noisy`` variants) — gradient attribution
* ``compute_scores`` — the broader SAD-style variant statistic family
  (SUM / logSUM / sqrtSUM / SAX / D1 / D2 / JS / logJS)
* ``dna_letter_at``, ``plot_seq_scores``, ``plot_coverage_tracks`` — sequence logos and coverage plots

``shorkie.viz.load_cov``
------------------------

Ground-truth coverage I/O, for comparing predictions against observed data.

* ``CovFace(cov_file)`` — reads bigWig, BED, or HDF5; ``.read(chrom, start, end)``
* ``read_coverage(file, chrom, start, end)``
* ``seq_norm(...)`` — bin per-nucleotide coverage into the model's 16 bp output bins,
  mirroring ``hound_data``'s target processing

``shorkie.data``
----------------

* ``bed_helper`` — ``extract_intervals``, ``get_exon_mask``, ``generate_beds``
* ``util`` — job-submission and parallel-execution helpers

.. note::

   Submodules are not auto-imported. Import them explicitly::

       from shorkie.models.ensemble import load_ensemble, make_input, logSED
       from shorkie.helpers.yeast_helpers import get_ism
       from shorkie.viz.load_cov import read_coverage
