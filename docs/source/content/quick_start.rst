Quick start
===========

Score a single variant with Shorkie, from nothing to a number. Runs on CPU.

Assumes :doc:`installation` is done (conda env active, package installed,
``config/paths.yaml`` copied).

1. Get the weights and the genome
---------------------------------

.. code-block:: bash

   bash data/download.sh --minimal                      # 8 Shorkie folds, ~0.46 GB
   bash data/download.sh --genome -u <your-gcp-project>  # R64 FASTA + GTF + .fai

Model weights are public. The genome lives in the requester-pays data bucket, so
``--genome`` needs a billing project — you pay only the egress, which is cents.
Every file is MD5-verified against ``data/manifest.json`` as it lands.

.. important::

   Use the released genome, not a fresh Ensembl or SGD download. Chromosome naming
   is load-bearing and differs between the two files — the FASTA uses
   ``chrI``…``chrXVI``, the GTF uses ``I``…``XVI``. Anything else fails to resolve.

2. Score a variant
------------------

.. code-block:: bash

   python minimal_example/run_shorkie_variant.py --model_dir ./my_shorkie

.. code-block:: text

   Tracks: 5215
   Loading 8-fold ensemble from ./my_shorkie...
   Gene YAL016C-B: I:124306-124492
   Predicting reference...
   Predicting alternate...

   ==================================================
     Variant  : chrI:124373 T>C
     Gene     : YAL016C-B
     logSED   : +0.0643
   ==================================================
     logSED > 0 → alt increases predicted expression
     logSED < 0 → alt decreases predicted expression

That is the built-in demo variant. ``--model_dir`` is the only required flag:
``params.json`` and ``sheet.txt`` ship beside the script, and the genome resolves
through ``shorkie.config``.

Loading 8 folds on CPU takes a couple of minutes.

3. Score your own variant
-------------------------

.. code-block:: bash

   python minimal_example/run_shorkie_variant.py \
       --model_dir ./my_shorkie \
       --chrom chrII --pos 350000 --ref A --alt G --gene YBR115C

``--gene`` must name a gene present in the GTF: logSED is summed over that gene's
output bins, so the score is always *the effect of this variant on this gene*.

What the number means
---------------------

.. math::

   \mathrm{logSED} = \log_2\!\left(\sum_{\text{alt bins}} + 1\right)
                   - \log_2\!\left(\sum_{\text{ref bins}} + 1\right)

A log2 ratio of predicted coverage summed over the gene body and averaged across
all 5,215 tracks. Positive means the alternate allele increases predicted
expression. It is the metric used for the paper's eQTL and MPRA benchmarks, so
scores are directly comparable to those results.

Interpret magnitudes relatively, not absolutely: rank variants against each other
(or against a matched negative set, as the eQTL benchmark does) rather than reading
a single value as an effect size.

Where to go next
----------------

* :doc:`shorkie_usage` — coverage prediction, per-track effects, the Python API
* :doc:`shorkie_lm_usage` — the language model: masked-base prediction, embeddings
* :doc:`finetuning` — train on your own RNA-seq, including a runnable mini-demo
* :doc:`reproducing_figures` — regenerate the paper's figures
