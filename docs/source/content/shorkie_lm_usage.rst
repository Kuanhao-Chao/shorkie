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
forced to 170 — the DNA channels plus the 166 species-identity channels.

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

Runnable notebooks
------------------

* `examples/1_lm_load_and_inference.ipynb
  <https://github.com/calico/shorkie-paper/blob/main/examples/1_lm_load_and_inference.ipynb>`_
  — load + masked-token prediction
* `examples/2_lm_embeddings.ipynb
  <https://github.com/calico/shorkie-paper/blob/main/examples/2_lm_embeddings.ipynb>`_
  — attention-layer embeddings + PCA

Both are committed with outputs, executed on CPU against the released checkpoint.
