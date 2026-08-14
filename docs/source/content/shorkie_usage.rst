Using Shorkie
=============

The main model: an 8-fold ensemble that predicts RNA-seq/ChIP coverage across 5,215
tracks, and scores variant effects with **logSED**. Everything on this page runs on
CPU (a GPU is faster but not required).

Prerequisites: :doc:`installation`, plus

.. code-block:: bash

   data/download.sh --models finetuned            # or --minimal
   data/download.sh --genome -u <your-gcp-project>

.. _load-shorkie:

1. Load the model
-----------------

The 8 folds are averaged at prediction time. ``load_ensemble`` handles the fold
layout, slices the model to your track set, and wraps each fold with
reverse-complement averaging.

.. code-block:: python

   import pandas as pd, pysam
   from shorkie import config
   from shorkie.models.ensemble import load_ensemble

   model_dir    = str(config.path("models.shorkie_finetuned"))
   MINIMAL      = config.repo_root() / "minimal_example"
   params_file  = str(MINIMAL / "params.json")   # architecture
   targets_file = str(MINIMAL / "sheet.txt")     # the 5215-track sheet

   targets_df   = pd.read_csv(targets_file, index_col=0, sep="\t")
   target_index = targets_df.index
   print("tracks:", len(target_index))           # -> tracks: 5215

   models = load_ensemble(model_dir, params_file, target_index, num_folds=8)
   print(f"loaded {len(models)}-fold Shorkie ensemble")

Loading all 8 folds takes a couple of minutes on CPU. To iterate faster, pass
``num_folds=1`` — predictions are noisier but the API is identical.

.. _predict-coverage:

2. Predict coverage
-------------------

Build the ``(16384, 170)`` input with ``make_input`` (DNA one-hot plus the species
channel), then average the folds:

.. code-block:: python

   from shorkie.models.ensemble import make_input, ensemble_predict

   fasta = pysam.Fastafile(str(config.path("genome.fasta")))

   x = make_input(fasta, "chrI", 70000, 70000 + 16384)
   y = ensemble_predict(models, x)
   print(y.shape)        # -> (1, 1, 896, 5215)

896 output bins cover the 16,384 bp window at 16 bp resolution. Index the last axis
with a row of the targets sheet to pull out one assay:

.. code-block:: python

   import matplotlib.pyplot as plt

   cov = y[0, 0]                       # (896, 5215)
   plt.plot(cov[:, 0])                 # track 0
   plt.xlabel("bin (16 bp)"); plt.ylabel("predicted coverage")

.. note::

   Chromosome naming is load-bearing: the released FASTA uses ``chrI``…``chrXVI``
   while the GTF uses ``I``…``XVI``. Use ``data/download.sh --genome``; a genome
   fetched straight from Ensembl or SGD will not match and lookups will fail.

.. _variant-effect:

3. Score a variant (logSED)
---------------------------

**logSED** is the variant-effect metric used in the paper's eQTL and MPRA
benchmarks:

.. math::

   \mathrm{logSED} = \log_2\!\left(\sum_{\text{alt bins}} + 1\right)
                   - \log_2\!\left(\sum_{\text{ref bins}} + 1\right)

summed over the gene-body output bins and averaged across tracks. Positive means the
alternate allele *increases* predicted expression.

Command line
^^^^^^^^^^^^

.. code-block:: bash

   python minimal_example/run_shorkie_variant.py \
       --model_dir ./my_shorkie \
       --chrom chrI --pos 124373 --ref T --alt C --gene YAL016C-B

.. code-block:: text

   ==================================================
     Variant  : chrI:124373 T>C
     Gene     : YAL016C-B
     logSED   : +0.0643
   ==================================================
     logSED > 0 → alt increases predicted expression
     logSED < 0 → alt decreases predicted expression

Only ``--model_dir`` is required — ``params.json`` and ``sheet.txt`` ship beside the
script, and the genome resolves through ``shorkie.config``.

In Python
^^^^^^^^^

.. code-block:: python

   from baskerville import gene as bgene
   from shorkie.models.ensemble import logSED, logSED_per_track

   txome = bgene.Transcriptome(str(config.path("genome.gtf")))
   gobj  = txome.genes["YAL016C-B"]

   # centre a window on the gene and map it onto the model's output bins
   m0    = models[0]
   mid   = gobj.midpoint()
   start = mid - 16384 // 2
   x_ref = make_input(fasta, "chrI", start, start + 16384)

   x_alt = ... # same window with the alt allele substituted in the DNA channels
   y_ref, y_alt = ensemble_predict(models, x_ref), ensemble_predict(models, x_alt)

   gene_slice = gobj.output_slice(start, m0.target_lengths[0],
                                  m0.model_strides[0], span=False)
   print(logSED(y_ref, y_alt, gene_slice))          # scalar, averaged over tracks
   print(logSED_per_track(y_ref, y_alt, gene_slice)) # (5215,) per-track effects

``logSED_per_track`` is how you ask *which assays* a variant affects — rank by
``abs()`` and join against the targets sheet.

Runnable notebooks
------------------

* `examples/3_shorkie_load_and_predict.ipynb
  <https://github.com/calico/shorkie-paper/blob/main/examples/3_shorkie_load_and_predict.ipynb>`_
  — load + predict + plot a track
* `examples/4_shorkie_variant_effect.ipynb
  <https://github.com/calico/shorkie-paper/blob/main/examples/4_shorkie_variant_effect.ipynb>`_
  — logSED end-to-end plus per-track ranking

Both are committed with their outputs, executed on CPU against the released weights.

Where this is used in the paper
-------------------------------

The same ``load_ensemble`` / ``make_input`` / ``ensemble_predict`` / ``logSED`` API
scores the eQTL benchmark (Figure 7) and the MPRA benchmark (Figure 6) — see
:doc:`reproducing_figures`.
