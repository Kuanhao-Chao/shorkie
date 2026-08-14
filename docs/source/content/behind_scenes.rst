How Shorkie was trained
=======================

Shorkie is built in two stages: pretrain a masked DNA language model on many fungal
genomes, then fine-tune it into a supervised coverage predictor for *S. cerevisiae*.
A third model — trained from scratch on the identical data — measures what the
pretraining actually bought.

.. code-block:: text

   165 fungal genomes  ──(masked LM)──>  Shorkie_LM
                                              │  --restore trunk
                                              ▼
             5215 tracks  ──(supervised)──>  Shorkie
                                              ▲
                          (no --restore)      │
             5215 tracks  ──(supervised)──>  Shorkie_Random_Init

Stage 1 — the pretraining corpus
--------------------------------

Four corpora were built at increasing phylogenetic breadth. Shorkie_LM is trained
on the **165_Saccharomycetales** tier; the other three back the LM ablations.

.. list-table::
   :header-rows: 1
   :widths: 30 14 18 20 18

   * - Tier
     - Genomes
     - Train seqs
     - Valid seqs
     - Test seqs
   * - R64
     - 1
     - 1,201
     - 518
     - 528
   * - 80_strains
     - 80
     - 102,315
     - 518
     - 528
   * - **165_Saccharomycetales** ⭐
     - 165
     - 385,551
     - 518
     - 528
   * - 1341_Fungus
     - 1,361
     - 625,355
     - 518
     - 528

All four share the same held-out split, drawn from *S. cerevisiae* R64 only, by
whole chromosome:

* **valid** — chrXI, chrXIII, chrXV
* **test** — chrXII, chrXIV, chrXVI
* chrXI–XVI are excluded from training in every tier

Holding out whole chromosomes (rather than random windows) is what keeps the
evaluation honest: no train sequence overlaps a test region.

Sequences are 16,384 bp, repeat-masked, and encoded as ZLIB TFRecords. The species
lists are committed in ``data/species_lists/`` so the corpus build is reproducible;
the pipeline is `scripts/01_data_build/lm_corpus/
<https://github.com/calico/shorkie-paper/tree/main/scripts/01_data_build/lm_corpus>`_.

.. note::

   The public label ``1341_Fungus`` is historical — the cleaned corpus actually
   contains 1,361 assemblies, which is the authoritative number.

Stage 2 — language-model pretraining
------------------------------------

Standard masked-language-model training over DNA: mask 15% of positions, predict
the missing bases.

.. list-table::
   :widths: 34 66

   * - Driver
     - ``hound_train.py`` (single fold)
   * - Objective
     - ``loss = mlm``, ``mask_rate = 0.15``, ``use_bert = true``
   * - Learning rate
     - 1e-4
   * - Initialisation
     - random
   * - Output
     - 1 × ``model_best.h5`` → **Shorkie_LM**

The input is ``(16384, 170)``: four DNA channels plus 166 species-identity
channels. That species channel is what lets one model absorb 165 genomes without
confusing them — at inference for *S. cerevisiae*, column 114 is set.

Recipe: `scripts/02_train/shorkie_lm/
<https://github.com/calico/shorkie-paper/tree/main/scripts/02_train/shorkie_lm>`_.

Stage 3 — supervised fine-tuning
--------------------------------

The pretrained trunk is restored and a fresh regression head is trained over 5,215
coverage tracks, in an 8-fold cross-validation split. The 8 folds become the
released ensemble, averaged at prediction time.

.. list-table::
   :widths: 34 66

   * - Driver
     - ``westminster_train_folds.py --restore <LM>`` → ``hound_train.py`` per fold
   * - Objective
     - ``loss = poisson_mn``
   * - Learning rate
     - 2e-5
   * - Initialisation
     - **from Shorkie_LM**
   * - Resolution
     - 16 bp bins → 896 output bins per 16,384 bp window
   * - Output
     - 8 × ``model_best.h5`` → **Shorkie**

The tracks
^^^^^^^^^^

* **Induction Dynamics Gene Expression Atlas (IDEA)** — RNA-seq induction
  time-course samples, generated for this study by Calico Life Sciences
  (related to IDEA 1.0; Hackett *et al.*, *Mol Syst Biol*, 2020)
* **Yeast strain RNA-seq** — across diverse *S. cerevisiae* isolates
  (Caudal *et al.*, *Nat Genet*, 2024)
* **ChIP-exo** and **ChIP-MNase** (Rossi *et al.*, *Nature*, 2021)

Recipe: `scripts/02_train/shorkie_finetuned/
<https://github.com/calico/shorkie-paper/tree/main/scripts/02_train/shorkie_finetuned>`_.

Stage 3b — the ablation
-----------------------

Shorkie_Random_Init trains the **same** 8 folds on the **same** data with the
**same** architecture — just without the pretrained weights.

.. list-table::
   :header-rows: 1
   :widths: 34 33 33

   * -
     - Shorkie
     - Shorkie_Random_Init
   * - ``--restore``
     - **yes** (Shorkie_LM)
     - **no**
   * - ``params.json`` → ``task``
     - ``fine-tune``
     - ``supervised``
   * - ``params.json`` → ``learning_rate``
     - 2e-5
     - **5e-4**
   * - ``params.json`` → ``model`` block
     - — identical —
     - — identical —

Exactly two ``train`` fields differ, and the architecture block is byte-identical.
Random init needs the larger step because it has no pretrained weights to perturb
gently. Everything else — folds, data, warmup, optimizer betas — is held fixed, so
the difference in performance is attributable to pretraining.

.. note::

   The committed configs under ``scripts/02_train/`` match the released
   ``params.json`` files field-for-field, and a test in the repository pins that
   equality — so the published recipe cannot drift from the published weights.

Recipe: `scripts/02_train/shorkie_scratch/
<https://github.com/calico/shorkie-paper/tree/main/scripts/02_train/shorkie_scratch>`_.

Reproducing the training
------------------------

The end-to-end pipelines are staged ``00_setup → 01_data_build → 02_train →
03_eval → 04_analysis`` in `scripts/
<https://github.com/calico/shorkie-paper/tree/main/scripts>`_. Every script resolves
paths through ``config/paths.yaml`` and accepts ``--dry-run`` to print the fully
resolved command without launching it.

You do not need any of this to *use* Shorkie — download the released weights
instead. See :doc:`finetuning` to adapt the recipe to your own tracks.
