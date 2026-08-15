FAQ
===

Conceptual questions about what Shorkie is and how to use it. For **error messages**, see
:doc:`troubleshooting`.

----

Choosing and understanding the models
-------------------------------------

Which model should I use?
^^^^^^^^^^^^^^^^^^^^^^^^^

**Shorkie** — the 8-fold fine-tuned ensemble — unless you have a specific reason otherwise. It is the
model the paper's results are built on, and the one that predicts expression.

The other two answer different questions. **Shorkie_LM** gives representations and masked-base
probabilities, not expression. **Shorkie_Random_Init** is an experimental control that quantifies what
pretraining contributed — it is not a newer or better model, and because it never saw the LM it is
generally weaker.

→ :doc:`models` for all three, with checksums and direct links.

What is the difference between Shorkie and Shorkie_LM?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Shorkie *is* Shorkie_LM plus a supervised head. The LM is pretrained on 165 fungal genomes to predict
masked bases; Shorkie restores that trunk and trains a regression head over 5,215 coverage tracks. That
single ``--restore`` is the whole relationship — and dropping it is exactly what produces
Shorkie_Random_Init.

→ :doc:`behind_scenes` for the training pipeline · :doc:`finetuning` to do it on your own tracks.

Why does the input have 170 channels?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

4 DNA one-hot channels plus 166 species-identity channels. The species channel is what lets one model
absorb 165 genomes without conflating them; at inference for *S. cerevisiae*, column 114 is set.

The width is corpus-specific — ``4 + num_species + 1`` — so the tier LM variants range from 6 to 1366.
Using the wrong value is the most common failure mode.

→ :ref:`the num_features table <axes-dont-match>` · :doc:`shorkie_usage`

What are the extra LM checkpoints, and can I swap one in for Shorkie_LM?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Mostly no. Four of them (``*__unet_small``) are the corpus-scaling comparison behind Figure 1F/G — a
*different architecture* from the released Shorkie_LM, meaningful compared against each other rather than
against it. The one genuine drop-in alternative is ``1341_Fungus__unet_small_bert_drop``: same
architecture, trained on the largest corpus.

→ :doc:`models`

Running the models
------------------

Do I need a GPU?
^^^^^^^^^^^^^^^^

Not for inference. Loading the 8 folds and scoring a variant takes a couple of minutes on CPU, and
Figures 6–7 reproduce from released data on CPU. A GPU is needed for training and fine-tuning, and for a
few explicitly-marked figure panels.

→ :doc:`installation` for the exact requirements.

What does logSED mean, and how should I interpret the number?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A log2 ratio of predicted coverage summed over the gene body and averaged across tracks; positive means
the alternate allele increases predicted expression.

Interpret it **relatively, not absolutely** — rank variants against each other, or against a matched
negative set as the eQTL benchmark does, rather than reading a single value as an effect size.

→ :doc:`shorkie_usage` for the formula and per-track effects.

How do I score many variants efficiently?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Load the ensemble once and loop — loading dominates the cost, not prediction.
``reproduction/figure_07/panels/`` shows the batched pattern used for the paper's own eQTL benchmark.

→ :doc:`shorkie_usage`

Which assays does a variant affect?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Use ``logSED_per_track`` instead of ``logSED``: it returns one value per track, which you rank by
``abs()`` and join against the targets sheet.

→ :doc:`shorkie_usage` · :doc:`api`

Scope and limitations
---------------------

Can I use Shorkie on another species?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The supervised model is trained on *S. cerevisiae* tracks with a species channel fixed to *S. cerevisiae*,
so predictions for other species aren't supported. Shorkie_LM saw 165 *Saccharomycetales* genomes and is
more portable for representation work, but the paper does not evaluate it that way. Fine-tuning on your
own species' tracks is the supported route.

→ :doc:`finetuning`

Can I change the sequence length?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

No — 16,384 bp is baked into the architecture. Change it and the pretrained trunk no longer transfers.

Can I fine-tune on my own RNA-seq?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Yes, and that's the intended way to adapt Shorkie. Build coverage tracks and TFRecords, point
``datasets.supervised_data`` at them, and restore from ``models.shorkie_lm_checkpoint``. The head is sized
from your track count, so a different number of tracks needs no code change.

Try ``examples/6_finetune_minidemo.sh`` first — it runs the real pipeline on a tiny slice in about a
minute, before you commit GPU-days.

→ :doc:`finetuning`

Data and access
---------------

Why do downloads ask for a Google Cloud project?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The **model weights are fully public** and need no account. The **datasets** are on a requester-pays
bucket, so the downloader needs a billing-enabled project via ``-u PROJECT``; you pay egress only, which
is cents for most artifacts.

→ :doc:`data_resources`

I can't use Google Cloud at all. Can I still get the data?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Yes — this is a supported case, not a workaround. Requester-pays needs a billing-enabled Google Cloud
project, which is impractical in some regions, so the models and pretraining corpora are also published
independently of Google infrastructure.

→ :doc:`data_resources`

Which corpus was Shorkie_LM actually trained on?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**165_Saccharomycetales** (165 genomes, 385,551 training sequences). The other three tiers exist to
support the corpus-scaling ablation.

→ :doc:`behind_scenes` · :doc:`data_resources`

If I train my own model on these corpora, what should I keep?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The **held-out split**. All four tiers share one split drawn from *S. cerevisiae* R64 only and split by
whole chromosome — valid: chrXI, chrXIII, chrXV; test: chrXII, chrXIV, chrXVI — with chrXI–XVI excluded
from training everywhere. Re-splitting randomly makes your numbers incomparable to ours, and leaks
sequence between train and test.

→ :doc:`data_resources`

Reproducibility
---------------

How do I know the reproduction is faithful?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Every figure ships a ``verify_figNN.csv`` comparing published against regenerated numbers at
``rtol=0.02`` — **206 of 206 checks pass** — plus an independent harness that re-derives headline numbers
straight from the eval artifacts (8/8), and a log of a fresh headless re-execution producing
byte-identical results.

→ :doc:`reproducing_figures` · :doc:`gallery`

Can I reproduce the figures without the training cluster?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Figures 6 and 7 fully, on CPU, from released data. The others need a gated intermediate produced by the
cited ``scripts/`` stage.

→ :doc:`reproducing_figures`

How do I cite Shorkie?
^^^^^^^^^^^^^^^^^^^^^^

Cite the preprint, and the original data sources if you use the benchmark or training data.

→ :doc:`citation`

----

Still stuck? Check :doc:`troubleshooting`, then `open an issue
<https://github.com/calico/shorkie-paper/issues>`__.
