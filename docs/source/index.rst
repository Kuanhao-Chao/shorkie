.. _main:

Shorkie
=======

**Predicting dynamic expression patterns in budding yeast with a fungal DNA language model.**

Shorkie is a sequence-to-expression model for *Saccharomyces cerevisiae*: a masked DNA
language model pretrained on hundreds of fungal genomes, then fine-tuned on 5,215
epigenomic and transcriptomic tracks — including a large set of transcriptional-regulator
induction RNA-seq experiments generated for this study — to predict RNA-seq coverage and
variant effects.

.. image:: https://img.shields.io/badge/License-Apache_2.0-blue.svg
   :target: https://www.apache.org/licenses/LICENSE-2.0
   :alt: License

.. image:: https://img.shields.io/badge/bioRxiv-10.1101%2F2025.09.19.677475-b31b1b.svg
   :target: https://doi.org/10.1101/2025.09.19.677475
   :alt: Preprint

----

What you can do with Shorkie
----------------------------

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: 🧬 Predict RNA-seq coverage
      :link: content/shorkie_usage
      :link-type: doc

      Run the 8-fold ensemble over any 16 kb window and get predicted coverage
      across all 5,215 tracks.

   .. grid-item-card:: 🔬 Score variant effects
      :link: content/shorkie_usage
      :link-type: doc

      Compute **logSED** for a SNP — the metric used in the paper's eQTL and MPRA
      benchmarks — from the command line or a notebook.

   .. grid-item-card:: 🧠 Use the DNA language model
      :link: content/shorkie_lm_usage
      :link-type: doc

      Masked-token prediction and self-attention embeddings from the fungal LM
      that Shorkie is built on.

   .. grid-item-card:: 🎛️ Fine-tune on your own tracks
      :link: content/finetuning
      :link-type: doc

      Transfer the LM trunk to your own RNA-seq / ChIP data, including a small
      demo you can actually run.

   .. grid-item-card:: 📊 See the analysis
      :link: content/gallery
      :link-type: doc

      A figure-by-figure tour of the paper, each linked to an executed notebook
      that regenerates it and re-checks the published numbers.

   .. grid-item-card:: 💾 Get the data
      :link: content/data_resources
      :link-type: doc

      Every dataset curated for this study — models, four pretraining corpora,
      reference genome, and the eQTL/MPRA benchmarks — with sizes and commands.

----

Quick start
-----------

.. code-block:: bash

   git clone --recurse-submodules https://github.com/calico/shorkie-paper.git
   cd shorkie-paper
   conda env create -f environment.yml && conda activate yeast_ml
   pip install -e external/baskerville-yeast -e external/westminster -e .
   cp config/paths.example.yaml config/paths.yaml

   bash data/download.sh --minimal                     # 8 Shorkie folds (~0.46 GB)
   bash data/download.sh --genome -u <your-gcp-project> # R64 FASTA + GTF

   python minimal_example/run_shorkie_variant.py \
       --model_dir ./my_shorkie

See :doc:`content/installation` for details and :doc:`content/quick_start` for a
walkthrough of that command's output.

----

The three released models
-------------------------

.. list-table::
   :header-rows: 1
   :widths: 22 20 58

   * - Model
     - What it is
     - Use it for
   * - **Shorkie_LM**
     - Masked DNA language model, pretrained on 165 *Saccharomycetales* genomes
     - Masked-base prediction, sequence embeddings, and as the starting point for fine-tuning
   * - **Shorkie**
     - 8-fold supervised ensemble, fine-tuned from Shorkie_LM on 5,215 tracks
     - Coverage prediction and variant-effect scoring — this is the main model
   * - **Shorkie_Random_Init**
     - Same architecture and data, trained from scratch (no LM pretraining, lr 5e-4)
     - The ablation that isolates what LM pretraining contributes

All three are public and catalogued with checksums — see :doc:`content/models`.

----

.. toctree::
   :maxdepth: 2
   :caption: Getting started

   content/installation
   content/quick_start
   content/models

.. toctree::
   :maxdepth: 2
   :caption: Using Shorkie

   content/shorkie_usage
   content/shorkie_lm_usage
   content/finetuning

.. toctree::
   :maxdepth: 2
   :caption: Background

   content/gallery
   content/behind_scenes
   content/data_resources
   content/reproducing_figures

.. toctree::
   :maxdepth: 2
   :caption: Reference

   content/api
   content/faq
   content/troubleshooting
   content/citation
   content/license
   content/contact
