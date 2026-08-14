Data and resources
==================

Everything released for Shorkie is catalogued — with sizes, MD5s, and destination
paths — in a single machine-readable file, `data/manifest.json
<https://github.com/calico/shorkie-paper/blob/main/data/manifest.json>`_.
``data/download.sh`` is driven entirely by it, and
``scripts/00_setup/verify_release.py`` audits the buckets against it.

Two buckets
-----------

.. list-table::
   :header-rows: 1
   :widths: 26 16 58

   * - Bucket
     - Access
     - Holds
   * - ``gs://seqnn-share``
     - **Public**
     - Model weights, under the ``shorkie_models/`` prefix. Plain HTTPS works.
   * - ``gs://shorkie-paper``
     - **Requester-pays**
     - Genome, corpora, supervised tracks, benchmark scores. Needs ``gsutil`` and
       a billing project (``-u PROJECT``); you pay egress only.

.. code-block:: bash

   data/download.sh --models all                        # public, no -u needed
   data/download.sh --genome     -u <your-gcp-project>
   data/download.sh --eqtl       -u <your-gcp-project>
   data/download.sh --mpra all   -u <your-gcp-project>
   data/download.sh --supervised [bigwigs|tfrecords|all] -u <your-gcp-project>
   data/download.sh --lm-corpus  <tier|all> -u <your-gcp-project>

Add ``--dry-run`` to any of these to see exactly what would be fetched.

Reference genome
----------------

``data/download.sh --genome`` → ``<release_root>/genome/R64/``

.. list-table::
   :header-rows: 1
   :widths: 40 14 46

   * - File
     - Size
     - Notes
   * - ``GCA_000146045_2.cleaned.fasta``
     - 11.7 MB
     - Chromosomes named ``chrI``…``chrXVI``. A ``.fai`` index is built on download.
   * - ``GCA_000146045_2.59.gtf``
     - 10.0 MB
     - Ensembl Fungi release 59. Sequences named ``I``…``XVI`` (plus ``Mito``).

.. warning::

   The two files use **different** chromosome naming, and the code depends on it.
   Use these released copies — a genome fetched straight from Ensembl or SGD will
   not match, and every example will fail to find its sequence.

   The released FASTA is the DUST-softmasked variant (892,009 repeat bases
   lowercased). It is equivalent for modelling: ``baskerville.dna.dna_1hot()``
   uppercases before encoding, so predictions are identical.

Pretraining corpora
-------------------

Four tiers, each with raw genomes and matched 16,384 bp ZLIB TFRecords, under
``gs://shorkie-paper/data/unsupervised/{genome,processed}/<tier>/``. Sizes run to
tens of GB. Sequence counts and the chromosome-holdout split are in
:doc:`behind_scenes`; the species lists are committed in ``data/species_lists/``.

Supervised tracks
-----------------

The 5,215-track dataset on S288C R64-3-1 at 16 bp resolution:

* ``gs://shorkie-paper/data/supervised/bigwigs/`` — coverage tracks (~93 GB)
* ``gs://shorkie-paper/data/supervised/processed/`` — 8-fold TFRecords (~10 GB,
  ~1.3 GB per fold) plus ``statistics.json``, ``sequences.bed``, ``targets.txt``

The targets sheet is also committed at ``minimal_example/sheet.txt`` so you can
inspect the track metadata without downloading anything.

Benchmark data
--------------

Reproduction-minimal subsets: enough to regenerate Figures 6 and 7 on CPU without
re-scoring anything.

**cis-eQTL** (Figure 7) — ``gs://shorkie-paper/eqtl/``

* ``scores/`` (~51 MB) — per-SNP scores for Shorkie / Shorkie_LM / Shorkie_Random_Init
* ``dream_eval/`` (~13 MB) — DREAM-model baselines

Three independent resources are benchmarked:

.. list-table::
   :header-rows: 1
   :widths: 26 14 60

   * - Study
     - Variants
     - Reference
   * - Caudal *et al.*
     - 1,901
     - Pan-transcriptome across ~1,000 isolates, *Nat Genet* 56, 1278–1287 (2024)
   * - Kita *et al.*
     - 683
     - High-resolution *cis*-regulatory mapping, *PNAS* 114 (2017)
   * - Renganaath *et al.*
     - 142
     - MPRA-validated core-promoter variants, *eLife* 9, e62669 (2020)

**MPRA** (Figure 6) — ``gs://shorkie-paper/mpra/``

* ``ground_truth/`` — MAUDE expression for the held-out set
* ``test_subset_ids/`` (~21 MB), ``scores/`` (~1.6 GB) — cached Shorkie logSED
* ``dream/`` — DREAM-RNN baseline output

From the Random Promoter DREAM Challenge (Rafi *et al.*, *Nat Biotechnol*, 2024):
71,103 held-out promoters spanning native, random, high/low-expression,
"challenging", SNV, motif-perturbation and motif-tiling categories.

Not re-hosted
-------------

Large third-party inputs are documented in the manifest (``external_raw``) but
obtained from their original sources:

* the 1011-yeast-genomes gVCF (~51 GB) — `1002 Yeast Genomes
  <http://1002genomes.u-strasbg.fr/files/>`_
* the full DREAM Challenge 2022 sequences and PrixFixe/DREAM-RNN weights —
  `Synapse syn28469146 <https://www.synapse.org/#!Synapse:syn28469146>`_

Verifying a download
--------------------

.. code-block:: bash

   python scripts/00_setup/verify_release.py -u <your-gcp-project> --strict

Checks every catalogued artifact against the buckets — size and MD5 for models,
non-empty prefixes for datasets. Model downloads are MD5-verified as they land;
large dataset prefixes rely on gsutil's transfer integrity check.
