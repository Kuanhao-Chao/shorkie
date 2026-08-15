Troubleshooting
===============

Organised by the **error message you actually see**. Each entry gives the cause, the fix, and a link to
the page that explains it properly.

If nothing here matches, please `open an issue
<https://github.com/calico/shorkie-paper/issues>`__ with the command you ran, the full traceback, and how
you obtained the weights.

----

Installation and setup
----------------------

``fatal: Could not read from remote repository`` when cloning
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: text

   Cloning into '.../external/baskerville-yeast'...
   fatal: Could not read from remote repository.
   Please make sure you have the correct access rights and the repository exists.
   fatal: clone of 'git@github.com:calico/baskerville-yeast.git' into submodule path ... failed

**Cause.** Older checkouts pinned the submodules to SSH URLs, so ``--recurse-submodules`` tried SSH even
when you cloned over HTTPS. Without GitHub SSH keys it fails and ``external/`` is left empty.

**Fix.** This is fixed on ``main`` — the submodules now use HTTPS. If you have an older clone:

.. code-block:: bash

   git pull
   git submodule sync --recursive
   git submodule update --init --recursive

Both repositories (`baskerville-yeast <https://github.com/calico/baskerville-yeast>`__ and
`westminster <https://github.com/calico/westminster>`__) are public, so no credentials are needed.

``ModuleNotFoundError: No module named 'baskerville'``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The submodules exist but weren't installed. Order matters — model code first:

.. code-block:: bash

   pip install -e external/baskerville-yeast -e external/westminster -e .

If ``external/`` is empty, see the previous entry. → :doc:`installation`

``ModuleNotFoundError: No module named 'shorkie'``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``pip install -e .`` wasn't run, or you're in a different environment than the one you installed into.
``conda activate yeast_ml``, then re-run it. → :doc:`installation`

----

Downloads
---------

``Bucket is a requester pays bucket but no user project provided``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Cause.** Everything except the model weights lives in ``gs://shorkie-paper``, which is requester-pays.

**Fix.** Pass a **billing-enabled** GCP project:

.. code-block:: bash

   data/download.sh --genome -u my-project

Being a member of a project isn't sufficient — billing must be enabled on it, or you get
``The billing account for the owning project is disabled``. Model weights need no ``-u`` at all;
they're on the public ``gs://seqnn-share``.

**No access to Google Cloud at all?** That's a supported case — see :doc:`data_resources`.

``md5 FAIL <file> (got … want …)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A truncated or corrupted download. Delete the file and re-run; ``download.sh`` verifies every model file
against ``data/manifest.json`` as it lands. Persistent failures on the same file are worth an issue.

``gsutil required for gs://… (requester-pays)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``gsutil`` isn't on ``PATH``. Install the Google Cloud SDK, or point at an existing install with
``export SHORKIE_GSUTIL=/path/to/gsutil``. Public model files fall back to ``wget``/``curl``; the
requester-pays datasets genuinely need ``gsutil``.

----

Loading models
--------------

.. _axes-dont-match:

``Error loading weights by name: axes don't match array``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**The single most common problem.** Shorkie does not take plain 4-channel DNA. Its input is
``(16384, 170)``: 4 DNA one-hot channels **plus 166 species-identity channels**. If you copy
``params.json`` and set ``num_features = 4``, the first convolution has the wrong shape and the weights
refuse to load.

**Fix.** Set ``num_features`` explicitly before constructing the model:

.. code-block:: python

   params["model"]["num_features"] = 170     # 4 DNA + 165 species + 1
   model = seqnn.SeqNN(params["model"])
   model.restore(checkpoint, trunk=False, by_name=False)

Better, let the helper build the input for you — it handles the species channel:

.. code-block:: python

   from shorkie.models.ensemble import make_input
   x = make_input(fasta, "chrI", start, start + 16384)   # -> (16384, 170)

.. important::

   ``num_features`` is **not recorded in params.json** — you must supply it, and it is corpus-specific:
   ``4 (DNA) + num_species + 1``.

   .. list-table::
      :header-rows: 1
      :widths: 40 20 20 20

      * - Model
        - Corpus
        - Species
        - ``num_features``
      * - **Shorkie** / **Shorkie_LM** / **Shorkie_Random_Init**
        - 165_Saccharomycetales
        - 165
        - **170**
      * - ``R64_yeast__unet_small``
        - R64
        - 1
        - 6
      * - ``80_strains__unet_small``
        - 80_strains
        - 80
        - 85
      * - ``1341_Fungus__unet_small`` / ``…_bert_drop``
        - 1341_Fungus
        - 1361
        - 1366

→ :doc:`shorkie_usage` · :doc:`models`

The model loads, but predictions look like noise
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Same root cause as above, failing silently. If you build a 4-channel input and pad the rest with zeros
the shapes match, so nothing errors — but the species channel is never set, and the model has never seen
such an input. Use ``make_input``, which sets column 114 for *S. cerevisiae*.

``FileNotFoundError: .../shorkie_lm/train/params.json``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The released layout puts ``params.json`` at the **model-dir root**, with only the checkpoint under
``train/``. Some author work-dirs put it under ``train/`` too, so accept either:

.. code-block:: python

   params_file = next(p for p in (f"{lm_dir}/params.json", f"{lm_dir}/train/params.json")
                      if os.path.exists(p))

→ :doc:`models`

Loading 8 folds is very slow, or runs out of memory
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Expected: the ensemble is 8 separate models and takes a couple of minutes on CPU. While iterating, use
``num_folds=1`` — noisier predictions, identical API. Load the ensemble **once** and loop over your
variants; loading dominates the cost. → :doc:`shorkie_usage`

----

Genome and annotation
---------------------

``KeyError`` / ``ValueError`` on a chromosome or gene name
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Almost always a naming mismatch. **The two reference files use different conventions, and this is
deliberate:**

.. list-table::
   :header-rows: 1

   * - File
     - Naming
   * - ``GCA_000146045_2.cleaned.fasta``
     - ``chrI`` … ``chrXVI``
   * - ``GCA_000146045_2.59.gtf``
     - ``I`` … ``XVI`` (plus ``Mito``)

A genome downloaded straight from Ensembl or SGD will not match. Use the released copies:

.. code-block:: bash

   data/download.sh --genome -u <your-gcp-project>

``--gene`` must also name a gene present in the GTF, since logSED is summed over that gene's output bins.
→ :doc:`data_resources`

``[E::fai_build3] Failed to open FASTA index``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``pysam`` needs a ``.fai`` beside the FASTA. ``download.sh --genome`` builds it for you; otherwise:

.. code-block:: bash

   samtools faidx GCA_000146045_2.cleaned.fasta

----

Data and TFRecords
------------------

Reading a TFRecord yields zero records, with no error
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The corpus TFRecords are **ZLIB-compressed**. Without the compression type TensorFlow returns an empty
dataset rather than raising:

.. code-block:: python

   ds = tf.data.TFRecordDataset(path, compression_type="ZLIB")   # required

→ :doc:`data_resources`

``1341_Fungus`` TFRecords aren't where the other tiers' are
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

That tier nests one level deeper, under a ``1342_Fungus/`` subdirectory — a different number — so
``--lm-corpus 1341_Fungus`` gives you
``…/processed/1341_Fungus/1342_Fungus/*.tfr`` while the other three land flat. A fixed-depth glob will
miss them. (The label is historical too: the cleaned corpus holds **1,361** assemblies.)
→ :doc:`data_resources`

----

Fine-tuning
-----------

``Skipping loading weights for layer … dense_22 … expects (384, 5215), received (384, 384)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**This is expected and means it's working.** ``--restore`` transfers the pretrained LM *trunk*; the
supervised head is new and has a different shape, so it is deliberately not restored. Seeing this warning
confirms fine-tuning is doing the right thing. → :doc:`finetuning`

Training loss barely moves / correlation near zero
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you're running ``examples/6_finetune_minidemo.sh``, that's the point — 8 training sequences and two
epochs cannot learn anything; it exists to prove the pipeline runs. For a real model use the production
recipe. On your own data, check that ``warmup_steps`` is sensible for your run length: the released value
is 5000, so a short run never leaves warmup. → :doc:`finetuning`

----

Reproducing figures
-------------------

A notebook says an input is missing and names a script to run
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Working as intended. Figures 6 and 7 run end-to-end from released data; the others consume a gated
intermediate (ISM/MoDISco ``.h5``, embeddings, training logs) that isn't in the public release. Run the
cited ``scripts/`` stage, then point the relevant ``results.*`` key in ``config/paths.yaml`` at your
output. → :doc:`reproducing_figures`

My reproduced number differs slightly from the paper
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

First check which weights you loaded. Documented values come from the **released** weights; earlier
versions of the repository resolved ``models.shorkie_finetuned`` to an author work-dir whose weights are
*not* the released ones, which gave slightly different scores. That's fixed, and a test now pins the
config to the release layout. ``num_folds`` matters too — documented values use all 8.
