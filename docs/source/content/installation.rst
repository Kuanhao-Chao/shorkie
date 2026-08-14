Installation
============

Shorkie's model code lives in two pinned forks — `baskerville-yeast
<https://github.com/calico/baskerville-yeast>`_ and `westminster
<https://github.com/calico/westminster>`_ — which ship as submodules of the
`shorkie-paper <https://github.com/calico/shorkie-paper>`_ repository. That repo
also provides an installable helper package (``shorkie``), the release catalogue,
the examples, and the figure notebooks.

Requirements
------------

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Component
     - Requirement
   * - OS
     - Linux
   * - Python
     - 3.9 (pinned by ``environment.yml``; the conda env is the supported path)
   * - TensorFlow
     - ~2.15
   * - CPU
     - Enough for inference, variant scoring, and reproducing the figures from released data
   * - GPU
     - Needed only for training / fine-tuning and a few GPU-marked figure panels.
       Those also need ``tensorrt==8.6.1`` and a CUDA-enabled TensorFlow build.

.. note::

   Everything on this site that is labelled *CPU* was verified on CPU from a clean
   download — no GPU and no cluster access required.

Install
-------

.. code-block:: bash

   git clone --recurse-submodules https://github.com/calico/shorkie-paper.git
   cd shorkie-paper

   conda env create -f environment.yml
   conda activate yeast_ml

   # model code first, then the helper package
   pip install -e external/baskerville-yeast -e external/westminster -e .

If you cloned without ``--recurse-submodules``:

.. code-block:: bash

   git submodule update --init

Configure paths
---------------

Every filesystem path in the repository resolves through ``config/paths.yaml`` —
there are no hardcoded machine paths in the pipeline code.

.. code-block:: bash

   cp config/paths.example.yaml config/paths.yaml

The defaults already point at the layout ``data/download.sh`` creates, so for
normal use you only need to set ``release_root`` (where downloads land). Resolution
order is ``$SHORKIE_CONFIG`` → ``config/paths.yaml`` → ``config/paths.example.yaml``.

.. code-block:: python

   from shorkie import config
   config.path("models.shorkie_finetuned")   # -> Path to the 8-fold ensemble
   config.path("genome.fasta")               # -> Path to the R64 FASTA

Containers
----------

``containers/`` ships a ``Dockerfile`` and an ``apptainer.def`` that build the
environment and both submodules, for a scheduler-free run.

Verify the install
------------------

.. code-block:: bash

   bash scripts/00_setup/verify_install.sh   # imports shorkie, resolves keys, runs pytest
   pytest -q                                 # release-integrity + smoke tests

Next: :doc:`quick_start`.
