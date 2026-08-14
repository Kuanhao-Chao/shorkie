Fine-tuning on your own RNA-seq
===============================

Shorkie *is* Shorkie_LM plus a supervised head. Fine-tuning on your own tracks is
the same operation the paper ran: restore the pretrained LM trunk, attach a fresh
regression head over your track set, and keep training.

.. code-block:: text

   Shorkie_LM  --restore-->  trunk (pretrained)  +  head (new, N tracks)  =  your model

Dropping ``--restore`` is exactly what produces Shorkie_Random_Init — that one flag
is the entire ablation.

Try it first: the mini-demo
---------------------------

Before committing GPU-days, run the mechanism on a tiny slice of the released data:

.. code-block:: bash

   bash examples/6_finetune_minidemo.sh -u <your-gcp-project>

It downloads ~96 MB (a byte-range prefix of one released fold shard), carves 16 real
sequences into an 8/4/4 train/valid/test split, shrinks the training schedule, and
runs the real ``hound_train.py --restore`` path.

.. warning::

   **The mini-demo's output is not a usable model.** Sixteen sequences and a couple
   of epochs cannot train anything. It exists to prove the pipeline runs end to end
   and to show which knobs matter. For a real model, use the production recipe below.

Add ``--dry-run`` to print every resolved command without executing anything.

What the demo changes
^^^^^^^^^^^^^^^^^^^^^

Only the knobs controlling how *long* training runs. The parts that define what
fine-tuning is — ``task``, ``loss``, optimizer, learning rate — are untouched:

.. list-table::
   :header-rows: 1
   :widths: 32 22 22 24

   * - Parameter
     - Released
     - Mini-demo
     - Why
   * - ``batch_size``
     - 8
     - 1
     - only 8 training sequences exist
   * - ``warmup_steps``
     - 5000
     - 1
     - the demo ends long before step 5000, so the LR would never warm up
   * - ``train_epochs_max``
     - 5000
     - 2
     - finish in minutes
   * - ``steps_per_epoch_max``
     - 500
     - 4
     - ditto
   * - ``patience``
     - 150
     - 1
     - ditto
   * - ``task`` / ``loss`` / ``learning_rate``
     - ``fine-tune`` / ``poisson_mn`` / 2e-5
     - *unchanged*
     - these define the recipe

The production recipe
---------------------

.. code-block:: bash

   # 8 folds, the full released supervised set, ~8 GPU-h per fold
   sbatch examples/5_finetune_lm_on_rnaseq.sh

   # or scheduler-free
   scripts/common/submit.sh --profile gpu examples/5_finetune_lm_on_rnaseq.sh

Under the hood:

.. code-block:: bash

   python westminster_train_folds.py \
       --restart -f 8 -e yeast_ml \
       --restore <Shorkie_LM model_best.h5> \
       --eval_dir <lm_corpus_split_root>/ \
       -o train -q a100 --rc --shifts "0,1" \
       params.json <supervised_data>

Everything resolves through ``config/paths.yaml``:

.. list-table::
   :header-rows: 1
   :widths: 34 66

   * - Config key
     - What it points at
   * - ``models.shorkie_lm_checkpoint``
     - the ``--restore`` target (Shorkie_LM ``model_best.h5``)
   * - ``datasets.supervised_data``
     - your 8-fold TFRecord directory
   * - ``datasets.lm_corpus_split_root``
     - ``--eval_dir``

Using your own tracks
---------------------

1. **Build coverage tracks.** Take your RNA-seq (or ChIP-exo/MNase) from FASTQ to
   BigWig with the pipeline in `scripts/01_data_build/supervised_tracks/
   <https://github.com/calico/shorkie-paper/tree/main/scripts/01_data_build/supervised_tracks>`_.

2. **Write a targets sheet.** A tab-separated file with one row per track —
   ``index, identifier, file, sum_stat, clip_soft, description, group``. The released
   5,215-track sheet (``minimal_example/sheet.txt``) is the reference format.

3. **Make TFRecords.** ``hound_data.py`` bins coverage into 16 bp windows over
   16,384 bp sequences and writes the fold shards, plus the ``statistics.json``
   descriptor the loader reads.

4. **Point config at them** (``datasets.supervised_data``) and run example 5.

The head is sized from ``statistics.json``'s ``num_targets``, so a different number
of tracks needs no code change — only the trunk transfers, and the head is new
regardless.

Practical notes
---------------

* **GPU required.** Inference runs fine on CPU; training does not, realistically.
* **Sequence length is fixed at 16,384 bp** by the architecture. Change it and the
  LM trunk no longer transfers.
* **The species channel matters.** Inputs are ``(16384, 170)``: 4 DNA channels plus
  166 species-identity channels, with column 114 set for *S. cerevisiae*. Keep using
  ``make_input`` and this is handled for you.
* **Start from the LM, not from Shorkie.** ``models.shorkie_lm_checkpoint`` is the
  intended ``--restore`` target; restoring the already-fine-tuned Shorkie carries a
  head shaped for the original 5,215 tracks.
