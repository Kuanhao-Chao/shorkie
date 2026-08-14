FAQ and troubleshooting
=======================

Which model should I use?
-------------------------

**Shorkie** — the 8-fold fine-tuned ensemble — unless you have a specific reason
otherwise. Shorkie_LM gives representations and masked-base probabilities, not
expression. Shorkie_Random_Init exists to quantify what pretraining contributed; it
is an experimental control, not a newer or better model.

Do I need a GPU?
----------------

Not for inference. Loading the 8 folds and scoring a variant takes a couple of
minutes on CPU. Reproducing Figures 6 and 7 from released data is CPU-only too.
A GPU is needed for training and fine-tuning, and for a few explicitly-marked
figure panels.

"No such file or directory: ...GCA_000146045_2.cleaned.fasta"
-------------------------------------------------------------

The reference genome has not been downloaded:

.. code-block:: bash

   bash data/download.sh --genome -u <your-gcp-project>

Or pass ``--fasta_file`` / ``--gtf_file`` explicitly.

My chromosome or gene isn't found
---------------------------------

Almost always a naming mismatch. The released FASTA uses ``chrI``…``chrXVI``; the
GTF uses ``I``…``XVI`` (plus ``Mito``). A genome downloaded straight from Ensembl or
SGD will not match. Use ``data/download.sh --genome``.

``--gene`` must also name a gene present in the GTF — logSED is summed over that
gene's output bins.

"Bucket is a requester pays bucket but no user project provided"
----------------------------------------------------------------

Everything except the model weights lives in a requester-pays bucket. Pass a
billing-enabled project with ``-u``:

.. code-block:: bash

   data/download.sh --genome -u my-project

The project must have billing enabled — being a member of a project is not enough.

"Skipping loading weights for layer ... dense_22 ... expects (384, 5215)"
-------------------------------------------------------------------------

**Expected during fine-tuning.** The pretrained LM trunk transfers; the supervised
head is new and has a different shape, so it is deliberately not restored. Seeing
this warning means ``--restore`` is doing exactly what it should.

Why did my logSED differ from the number in the docs?
-----------------------------------------------------

Check which weights you loaded. Documented values come from the **released**
weights (``data/download.sh``). Earlier versions of this repository resolved
``models.shorkie_finetuned`` to an author work-dir whose weights are *not* the
released ones, which produced slightly different scores — that is fixed, and a test
now pins the config to the release layout.

Small differences can also come from ``num_folds``: the documented values use all 8.

How do I score many variants?
-----------------------------

Load the ensemble once and loop — loading dominates the cost. See
:doc:`shorkie_usage`; ``reproduction/figure_07/panels/`` shows the batched pattern
used for the paper's eQTL benchmark.

Can I use Shorkie on another species?
-------------------------------------

The supervised model is trained on *S. cerevisiae* tracks and its input carries a
species-identity channel fixed to *S. cerevisiae* (column 114), so predictions for
other species are not supported. Shorkie_LM saw 165 *Saccharomycetales* genomes and
is more portable for representation work, but was not evaluated that way in the
paper. Fine-tuning on your own species' tracks is the supported route
(:doc:`finetuning`).

Can I change the sequence length?
---------------------------------

No — 16,384 bp is baked into the architecture. Change it and the pretrained trunk
no longer transfers.

Something else
--------------

Open a `GitHub issue <https://github.com/calico/shorkie-paper/issues>`_ — please
include the command, the full traceback, and how you obtained the weights.
