Datasets
========

Everything curated for this study, and how to get it. All of it is catalogued — with sizes, MD5s and
destinations — in `data/manifest.json
<https://github.com/calico/shorkie-paper/blob/main/data/manifest.json>`__, which is exactly what
``data/download.sh`` reads and what ``scripts/00_setup/verify_release.py`` audits the buckets against.

At a glance
-----------

.. list-table::
   :header-rows: 1
   :widths: 34 12 12 42

   * - Dataset
     - Size
     - Access
     - Fetch with
   * - :ref:`Model weights <ds-models>` — LM + Shorkie + Random_Init
     - 0.97 GB
     - public
     - ``data/download.sh --models all``
   * - :ref:`LM variants <ds-lm-variants>` — other corpus tiers
     - 286 MB
     - public
     - ``data/download.sh --models lm-variants``
   * - :ref:`R64 reference genome <ds-genome>` — FASTA + GTF
     - 22 MB
     - requester-pays
     - ``data/download.sh --genome -u PROJECT``
   * - :ref:`LM corpus — R64 <ds-corpora>`
     - 23 MB
     - requester-pays
     - ``data/download.sh --lm-corpus R64 -u PROJECT``
   * - :ref:`LM corpus — 80_strains <ds-corpora>`
     - 1.5 GB
     - requester-pays
     - ``data/download.sh --lm-corpus 80_strains -u PROJECT``
   * - :ref:`LM corpus — 165_Saccharomycetales <ds-corpora>` ⭐
     - 3.7 GB
     - requester-pays
     - ``data/download.sh --lm-corpus 165_Saccharomycetales -u PROJECT``
   * - :ref:`LM corpus — 1341_Fungus <ds-corpora>`
     - 42.5 GB
     - requester-pays
     - ``data/download.sh --lm-corpus 1341_Fungus -u PROJECT``
   * - :ref:`Supervised tracks <ds-supervised>` — BigWigs
     - ~93 GB
     - requester-pays
     - ``data/download.sh --supervised bigwigs -u PROJECT``
   * - :ref:`Supervised tracks <ds-supervised>` — TFRecords
     - ~10 GB
     - requester-pays
     - ``data/download.sh --supervised tfrecords -u PROJECT``
   * - :ref:`cis-eQTL benchmark <ds-eqtl>`
     - ~64 MB
     - requester-pays
     - ``data/download.sh --eqtl -u PROJECT``
   * - :ref:`MPRA benchmark <ds-mpra>`
     - ~1.6 GB
     - requester-pays
     - ``data/download.sh --mpra all -u PROJECT``

⭐ = the corpus Shorkie_LM was actually pretrained on.

Add ``--dry-run`` to any of these to see precisely what would be fetched, without fetching it.

Two buckets, two access models
------------------------------

.. list-table::
   :header-rows: 1
   :widths: 26 18 56

   * - Bucket
     - Access
     - Holds
   * - ``gs://seqnn-share``
     - **Public**
     - Model weights, under ``shorkie_models/``. Plain HTTPS works — no account, no ``-u``.
   * - ``gs://shorkie-paper``
     - **Requester-pays**
     - Genome, corpora, supervised tracks, benchmark scores. Needs ``gsutil`` and a
       **billing-enabled** GCP project via ``-u PROJECT``; you pay egress only.

.. note::

   **If you cannot use Google Cloud**, requester-pays is a hard barrier — it requires a billing-enabled
   project, which is impractical in some regions. The models and pretraining corpora are therefore also
   published independently of Google infrastructure, via
   ``scripts/00_setup/zenodo_upload.py``. If the mirror you need isn't up yet, please
   `open an issue <https://github.com/calico/shorkie-paper/issues>`__ — that is a supported request, not
   an imposition.

   Two things are always reachable regardless: the **species lists** with every NCBI accession
   (`data/species_lists/ <https://github.com/calico/shorkie-paper/tree/main/data/species_lists>`__) and
   the **corpus build pipeline** (``scripts/01_data_build/lm_corpus/``), which together let you rebuild
   any tier from public sources.

.. _ds-models:

Model weights
-------------

**0.97 GB total · public · no account needed.** Full detail, including per-fold direct links and MD5s,
is on :doc:`models`.

.. list-table::
   :header-rows: 1
   :widths: 30 12 16 42

   * - Model
     - Size
     - Folds
     - Bucket path
   * - Shorkie_LM
     - 55 MB
     - 1
     - ``gs://seqnn-share/shorkie_models/shorkie_lm/``
   * - **Shorkie**
     - 461 MB
     - 8
     - ``gs://seqnn-share/shorkie_models/shorkie/``
   * - Shorkie_Random_Init
     - 455 MB
     - 8
     - ``gs://seqnn-share/shorkie_models/shorkie_random_init/``

Every file is MD5-verified against the manifest as it downloads.

.. _ds-lm-variants:

LM checkpoints for the other corpus tiers
-----------------------------------------

**286 MB · public.** Language models pretrained on the *other* corpus tiers — the corpus-scaling
comparison behind Figure 1F/G, plus the largest-corpus model at the released architecture.

.. list-table::
   :header-rows: 1
   :widths: 40 20 22 18

   * - Variant
     - Corpus
     - Architecture
     - ``num_features``
   * - ``R64_yeast__unet_small``
     - R64
     - ``unet_small``
     - 6
   * - ``80_strains__unet_small``
     - 80_strains
     - ``unet_small``
     - 85
   * - ``165_Saccharomycetales__unet_small``
     - 165_Saccharomycetales
     - ``unet_small``
     - 170
   * - ``1341_Fungus__unet_small``
     - 1341_Fungus
     - ``unet_small``
     - 1366
   * - ``1341_Fungus__unet_small_bert_drop``
     - 1341_Fungus
     - ``unet_small_bert_drop``
     - 1366

.. warning::

   These are **not** drop-in replacements for the released Shorkie_LM, which is 165_Saccharomycetales +
   ``unet_small_bert_drop``. The four ``unet_small`` runs are a different architecture and are meaningful
   compared against *each other*. The one genuine alternative is ``1341_Fungus__unet_small_bert_drop``.

   ``num_features`` is **not stored in params.json** and must be set at load time — it is
   ``4 (DNA) + num_species + 1``. Getting it wrong raises
   :ref:`"axes don't match array" <axes-dont-match>`.

.. _ds-genome:

R64 reference genome
--------------------

**22 MB.** Required by ``examples/`` and ``minimal_example/``.

.. list-table::
   :header-rows: 1
   :widths: 44 12 44

   * - File
     - Size
     - Notes
   * - ``GCA_000146045_2.cleaned.fasta``
     - 11.7 MB
     - Chromosomes ``chrI``…``chrXVI``. A ``.fai`` index is built on download.
   * - ``GCA_000146045_2.59.gtf``
     - 10.0 MB
     - Ensembl Fungi release 59. Sequences ``I``…``XVI`` (plus ``Mito``).

.. warning::

   The two files use **different chromosome naming**, and the code depends on it. Use these released
   copies — a genome fetched straight from Ensembl or SGD will not match, and every lookup will fail.
   See :doc:`troubleshooting`.

   The released FASTA is the DUST-softmasked variant (892,009 repeat bases lowercased), which is
   equivalent for modelling: the encoder uppercases before one-hot encoding.

.. _ds-corpora:

Pretraining corpora
-------------------

Four tiers of increasing phylogenetic breadth, each with raw genome assemblies and matched 16,384 bp
TFRecords. **Shorkie_LM was pretrained on 165_Saccharomycetales**; the others back the ablations.

.. list-table::
   :header-rows: 1
   :widths: 30 12 16 14 14 14

   * - Tier
     - Genomes
     - Train seqs
     - Genomes
     - TFRecords
     - Total
   * - R64
     - 1
     - 1,201
     - 12 MB
     - 12 MB
     - 23 MB
   * - 80_strains
     - 80
     - 102,315
     - 968 MB
     - 533 MB
     - 1.5 GB
   * - **165_Saccharomycetales** ⭐
     - 165
     - 385,551
     - 1.97 GB
     - 1.73 GB
     - 3.7 GB
   * - 1341_Fungus
     - 1,361
     - 625,355
     - 39.3 GB
     - 3.15 GB
     - 42.5 GB

Paths follow ``gs://shorkie-paper/data/unsupervised/{genome,processed}/<tier>/``.

**The held-out split is shared across all four tiers** and drawn from *S. cerevisiae* R64 only, split by
**whole chromosome**:

- **valid** — chrXI, chrXIII, chrXV
- **test** — chrXII, chrXIV, chrXVI
- chrXI–XVI are excluded from training in every tier

Splitting by whole chromosome rather than random windows is what keeps the evaluation honest. If you use
these corpora as a baseline, **keep this split** — re-splitting randomly makes your numbers incomparable
to the paper's and leaks sequence between train and test.

.. important::

   TFRecords are **ZLIB-compressed**. Read them with
   ``tf.data.TFRecordDataset(path, compression_type="ZLIB")`` — without it TensorFlow returns an empty
   dataset rather than raising.

   **``1341_Fungus`` nests one level deeper** than the other three tiers, under a ``1342_Fungus/``
   subdirectory (a different number), so a fixed-depth glob will miss its records. The public label is
   historical too: the cleaned corpus contains **1,361** assemblies.

Per-tier species lists with NCBI accessions are committed at `data/species_lists/
<https://github.com/calico/shorkie-paper/tree/main/data/species_lists>`__, so any tier can be rebuilt
from public sources with ``scripts/01_data_build/lm_corpus/``.

.. _ds-supervised:

Supervised tracks
-----------------

The **5,215-track** dataset on S288C R64-3-1 at 16 bp resolution that Shorkie is fine-tuned on.

- ``gs://shorkie-paper/data/supervised/bigwigs/`` — coverage tracks, **~93 GB**
- ``gs://shorkie-paper/data/supervised/processed/`` — 8-fold TFRecords, **~10 GB** (~1.3 GB per fold),
  plus ``statistics.json``, ``sequences.bed`` and ``targets.txt``

Sources:

- **Induction Dynamics Gene Expression Atlas (IDEA)** — RNA-seq induction time-course samples, generated
  for this study by Calico Life Sciences (related to Hackett, S. R. *et al.*, *Mol Syst Biol*, 2020)
- **Yeast strain RNA-seq** across diverse *S. cerevisiae* isolates (Caudal, É. *et al.*, *Nat Genet*, 2024)
- **ChIP-exo** and **ChIP-MNase** (Rossi, M. J. *et al.*, *Nature*, 2021)

The targets sheet is also committed at `minimal_example/sheet.txt
<https://github.com/calico/shorkie-paper/blob/main/minimal_example/sheet.txt>`__, so you can inspect the
track metadata without downloading anything.

.. _ds-eqtl:

cis-eQTL benchmark
------------------

**~64 MB.** Enough to regenerate Figure 7 on CPU without re-scoring.

- ``gs://shorkie-paper/eqtl/scores/`` (~51 MB) — per-SNP scores for Shorkie / Shorkie_LM / Shorkie_Random_Init
- ``gs://shorkie-paper/eqtl/dream_eval/`` (~13 MB) — DREAM-model baselines

Three independent resources:

.. list-table::
   :header-rows: 1
   :widths: 22 12 66

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

.. _ds-mpra:

MPRA benchmark
--------------

**~1.6 GB.** Enough to regenerate Figure 6 on CPU.

- ``ground_truth/`` — MAUDE expression for the held-out set
- ``test_subset_ids/`` (~21 MB) and ``scores/`` (~1.6 GB) — cached Shorkie logSED
- ``dream/`` — DREAM-RNN baseline output

From the Random Promoter DREAM Challenge (Rafi, A. M. *et al.*, *Nat Biotechnol*, 2024): 71,103 held-out
promoters spanning native, random, high/low-expression, "challenging", SNV, motif-perturbation and
motif-tiling categories.

Not re-hosted
-------------

Large third-party inputs are documented in the manifest (``external_raw``) but obtained from source:

- the 1011-yeast-genomes gVCF (~51 GB) — `1002 Yeast Genomes
  <http://1002genomes.u-strasbg.fr/files/>`__
- the full DREAM Challenge 2022 sequences and PrixFixe/DREAM-RNN weights —
  `Synapse syn28469146 <https://www.synapse.org/#!Synapse:syn28469146>`__

Verifying a download
--------------------

.. code-block:: bash

   python scripts/00_setup/verify_release.py -u <your-gcp-project> --strict

Checks every catalogued artifact against the buckets — size and MD5 for models, non-empty prefixes for
datasets. Model downloads are MD5-verified as they land; large dataset prefixes rely on gsutil's transfer
integrity check.

Licensing
---------

Model weights and the data we derived (TFRecords, cached scores) are **CC BY 4.0**. The genome assemblies
are third-party public data (Ensembl Fungi release 59 / NCBI GenBank) redistributed for reproducibility,
under their original providers' terms. Benchmark datasets carry their own terms. Repository code is
Apache-2.0. → :doc:`license` · :doc:`citation`
