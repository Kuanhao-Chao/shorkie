Reproducing the paper figures
=============================

Each of the seven main-text figures has one notebook in `notebooks/
<https://github.com/calico/shorkie-paper/tree/main/notebooks>`_. A notebook either
runs end-to-end from released data or loads a gated intermediate produced by the
cited ``scripts/`` stage, then renders the panels by calling that figure's builders
under ``reproduction/figure_NN/``.

.. list-table::
   :header-rows: 1
   :widths: 10 46 22 22

   * - Fig
     - Notebook
     - Subject
     - Checks
   * - 1
     - ``fig01_fungal_lm_corpus_architecture``
     - Corpus + architecture
     - 12/12
   * - 2
     - ``fig02_lm_conserved_motifs``
     - LM-discovered motifs
     - 21/21
   * - 3
     - ``fig03_supervised_rnaseq_prediction``
     - RNA-seq prediction
     - 33/33
   * - 4
     - ``fig04_promoter_splicing_motifs``
     - Promoter + splicing motifs
     - 38/38
   * - 5
     - ``fig05_timecourse_tf_induction``
     - MSN2/MSN4 time course
     - 10/10
   * - 6
     - ``fig06_mpra_variant_effects``
     - MPRA variant effects
     - 26/26
   * - 7
     - ``fig07_eqtl_variant_effects``
     - cis-eQTL variant effects
     - 66/66

How verification works
----------------------

Reproduction is not "the picture looks similar". Every figure carries a
``reproduced/verify_figNN.csv`` comparing published numbers against regenerated
ones:

.. code-block:: text

   panel,metric,reported,reproduced,delta,rtol,atol,verdict

**206 of 206 numeric checks pass**, at ``rtol=0.02``. On top of that,
``reproduction/recheck/recompute_recheck.py`` independently re-derives a handful of
headline numbers straight from the on-disk eval artifacts (8/8 pass), and
``determinism.csv`` records a fresh headless re-execution of every notebook
producing byte-identical verify CSVs.

Running them
------------

.. code-block:: bash

   conda activate yeast_ml
   jupyter lab notebooks/

Figures 6 and 7 reproduce **on CPU** from the released benchmark data — fetch it
with ``data/download.sh --eqtl --mpra`` (see :doc:`data_resources`). Others need a
gated intermediate (ISM/MoDISco ``.h5``, embeddings, training logs) that is not in
the released manifest: run the cited upstream ``scripts/`` stage first, then point
the relevant ``results.*`` key in ``config/paths.yaml`` at your output. Each
notebook states its own "Reproduces / Upstream / Requires" up front, and degrades
gracefully with a message naming the exact command to run when an input is absent.

The per-figure builders, published crops, and reproduced panels live under
`reproduction/ <https://github.com/calico/shorkie-paper/tree/main/reproduction>`_.
