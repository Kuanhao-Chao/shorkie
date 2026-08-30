Using Shorkie_LM
================

Shorkie_LM is the masked DNA language model that Shorkie is built on: a single
checkpoint pretrained on 165 *Saccharomycetales* genomes with ``loss=mlm`` and a
15% mask rate. Use it when you want *representations* or *masked-base
probabilities* rather than expression predictions.

Everything here runs on CPU.

.. code-block:: bash

   data/download.sh --models lm
   data/download.sh --genome -u <your-gcp-project>

1. Load the model
-----------------

The LM is a single checkpoint, so it is loaded directly through
``baskerville.seqnn`` rather than the ensemble helper. ``num_features`` must be
forced to 170 = **4 DNA + 1 + 165 species**.

.. note::

   The fifth channel is not a species channel. Every corpus in this repository
   has ``num_features = n_species + 5`` (6 / 85 / 170 / 1366 for 1 / 80 / 165 /
   1361 species), and a one-genome corpus needing six features is impossible
   under "4 DNA + N species". ``N_SPECIES = 166`` in ``ensemble.py`` folds that
   extra channel into the species block. No code shipped here ever writes it, and
   it is zero on every inference path — the LM masks by zeroing the four DNA
   channels, not by setting a mask channel.

.. code-block:: python

   import os, json, pysam
   from baskerville import seqnn
   from shorkie import config

   lm_dir = str(config.path("models.shorkie_lm"))

   # released layout keeps params.json at the dir root; some work-dirs put it under train/
   params_file = next(p for p in (f"{lm_dir}/params.json", f"{lm_dir}/train/params.json")
                      if os.path.exists(p))

   params = json.load(open(params_file))
   params["model"]["num_features"] = 170

   model = seqnn.SeqNN(params["model"])
   model.restore(f"{lm_dir}/train/model_best.h5", trunk=False, by_name=False)
   keras_model = model.model      # ~13.7 M parameters

2. Masked-base prediction
-------------------------

Mask a position and ask the model what belongs there. ``make_input`` zeroes the
four DNA channels at ``mask_pos`` — the same masking used during pretraining.

.. code-block:: python

   import numpy as np
   from shorkie.models.ensemble import make_input

   fasta  = pysam.Fastafile(str(config.path("genome.fasta")))
   start  = 70000
   centre = 16384 // 2

   x = make_input(fasta, "chrI", start, start + 16384, seq_len=16384, mask_pos=centre)
   y = keras_model(x.numpy()[None, ...])

   probs = np.asarray(y)[0, centre, :4]
   probs = probs / probs.sum()
   print(dict(zip("ACGT", probs.round(3))))

   truth = fasta.fetch("chrI", start + centre, start + centre + 1).upper()
   print("argmax:", "ACGT"[int(probs.argmax())], " genome:", truth)

The model is not an oracle — on a randomly chosen position it often disagrees with
the reference. What makes it useful is that the *distribution* encodes sequence
constraint, which is what transfers when you fine-tune.

3. Sequence embeddings
----------------------

Tap an intermediate layer to get representations for downstream analysis. The
first self-attention layer is a good general-purpose choice:

.. code-block:: python

   import tensorflow as tf

   embed = tf.keras.Model(inputs=keras_model.input,
                          outputs=keras_model.get_layer("multihead_attention").output)

   x = make_input(fasta, "chrI", 70000, 70000 + 16384)
   emb = embed(x.numpy()[None, ...]).numpy()[0]
   print(emb.shape)      # -> (128, 384)

   from sklearn.decomposition import PCA
   coords = PCA(n_components=2).fit_transform(emb)

List ``[l.name for l in keras_model.layers]`` to choose a different depth. These
embeddings are what Figure 2's motif and t-SNE analyses are built on.

4. Fine-tune it
---------------

Turning Shorkie_LM into a coverage predictor for your own tracks is its own page:
:doc:`finetuning`.

Try it without installing anything
----------------------------------

`khchao.com/shorkie-lab/shorkie_lm <https://khchao.com/shorkie-lab/shorkie_lm/>`_
runs this checkpoint's predictions over fourteen annotated yeast windows in the
browser: masked-base distributions at every position, constraint drawn as
``2 - entropy``, and the masked-motif and annotation-enrichment analyses. It is
precomputed, so there is nothing to download.

Two things it makes concrete that are easy to get wrong here:

* **An unmasked forward pass is not a prediction.** The model can see the base it
  is scoring and largely copies it (97.8% argmax). The number to quote comes from
  masking — iteratively, at the 15% rate this checkpoint was trained with, which
  gives **43.0% argmax and perplexity 3.380**.
* **Masking a whole motif is a different task from the one it was trained on.**
  Pretraining masks 15% of positions *scattered*; a contiguous 10 bp hole removes
  the local context the model relies on, and it falls back to emitting the base
  composition prior.

Runnable notebooks
------------------

* `examples/1_lm_load_and_inference.ipynb
  <https://github.com/calico/shorkie-paper/blob/main/examples/1_lm_load_and_inference.ipynb>`_
  — load + masked-token prediction
* `examples/2_lm_embeddings.ipynb
  <https://github.com/calico/shorkie-paper/blob/main/examples/2_lm_embeddings.ipynb>`_
  — attention-layer embeddings + PCA

Both are committed with outputs, executed on CPU against the released checkpoint.
