Released models
===============

Three models are public. All live in the **public** bucket ``gs://seqnn-share``
under the ``shorkie_models/`` prefix, are downloadable over plain HTTPS, and are
catalogued with sizes and MD5 checksums in `data/manifest.json
<https://github.com/calico/shorkie-paper/blob/main/data/manifest.json>`_.

.. list-table::
   :header-rows: 1
   :widths: 26 12 20 42

   * - Model
     - Folds
     - Training
     - What it's for
   * - **Shorkie_LM**
     - 1
     - Masked LM (``loss=mlm``, lr 1e-4) on 165 *Saccharomycetales* genomes
     - Masked-base prediction, embeddings, and the ``--restore`` target for fine-tuning
   * - **Shorkie**
     - 8
     - Fine-tuned from Shorkie_LM (``task=fine-tune``, lr 2e-5) on 5,215 tracks
     - **The main model.** Coverage prediction and variant effects
   * - **Shorkie_Random_Init**
     - 8
     - Same data/architecture from random init (``task=supervised``, lr 5e-4)
     - The ablation isolating the contribution of LM pretraining

Download
--------

.. code-block:: bash

   data/download.sh --models all           # all three (~1.4 GB)
   data/download.sh --models lm            # Shorkie_LM only
   data/download.sh --models finetuned     # the 8-fold Shorkie
   data/download.sh --models random_init   # the ablation
   data/download.sh --minimal              # just the 8 Shorkie folds (~0.46 GB)

Every file is MD5-verified against the manifest as it downloads.

Direct links
------------

* **Shorkie_LM** — `model_best.h5
  <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie_lm/train/model_best.h5>`_
* **Shorkie** (``gs://seqnn-share/shorkie_models/shorkie/``) —
  `f0 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie/f0/model_best.h5>`_ |
  `f1 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie/f1/model_best.h5>`_ |
  `f2 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie/f2/model_best.h5>`_ |
  `f3 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie/f3/model_best.h5>`_ |
  `f4 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie/f4/model_best.h5>`_ |
  `f5 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie/f5/model_best.h5>`_ |
  `f6 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie/f6/model_best.h5>`_ |
  `f7 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie/f7/model_best.h5>`_
* **Shorkie_Random_Init** (``gs://seqnn-share/shorkie_models/shorkie_random_init/``) —
  `f0 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie_random_init/f0/model_best.h5>`_ |
  `f1 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie_random_init/f1/model_best.h5>`_ |
  `f2 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie_random_init/f2/model_best.h5>`_ |
  `f3 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie_random_init/f3/model_best.h5>`_ |
  `f4 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie_random_init/f4/model_best.h5>`_ |
  `f5 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie_random_init/f5/model_best.h5>`_ |
  `f6 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie_random_init/f6/model_best.h5>`_ |
  `f7 <https://storage.googleapis.com/seqnn-share/shorkie_models/shorkie_random_init/f7/model_best.h5>`_

On-disk layout
--------------

``data/download.sh`` writes the layout the loaders expect::

    <release_root>/models/
    ├── shorkie_lm/
    │   ├── params.json
    │   └── train/model_best.h5
    ├── shorkie_finetuned/
    │   ├── params.json
    │   ├── targets.txt                  # the 5215-track sheet
    │   └── train/f{0..7}c0/train/model_best.h5
    └── shorkie_random_init/
        ├── params.json
        └── train/f{0..7}c0/train/model_best.h5

.. note::

   ``params.json`` sits at the **model-dir root**, while the checkpoint is under
   ``train/``. Some author work-dirs put ``params.json`` under ``train/`` too, so the
   examples accept either location.

Architecture
------------

All three share one ``unet_small_bert_drop`` architecture — the ablation deliberately
holds it fixed:

* **Input**: ``(16384, 170)`` — channels 0–3 are DNA one-hot, channels 4–169 are
  species identity (column 114 = *S. cerevisiae*).
* **Output**: ``(1, 1, 896, 5215)`` — 896 bins of 16 bp, one channel per track.
* ~13.7 M parameters.

The committed training configs under `scripts/02_train/
<https://github.com/calico/shorkie-paper/tree/main/scripts/02_train>`_ match the
released ``params.json`` field-for-field; a test in the repository pins that
equality so the published recipe cannot drift from the published weights.

Choosing a model
----------------

Use **Shorkie** unless you have a specific reason not to. Shorkie_Random_Init exists
to answer "how much did pretraining help?" — it is not a better or newer model, and
because it never saw the LM it is generally weaker. Shorkie_LM is the right choice
only when you want representations or masked-base probabilities rather than
expression predictions.
