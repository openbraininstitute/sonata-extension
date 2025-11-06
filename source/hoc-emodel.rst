HOC E-Model Templates
=====================

When instantiating a morphology in NEURON, the, `HOC`_ language is used.
For a model to be able to be used by `bluecellulab`_ or `neurodamus`_ a minimum set of public members are required.
This document outlines the requirements from the point of view of required attributes, and the functions that are called.

Functions:
----------

=============== =============== ========================================================================
Name            Arguments       Description
=============== =============== ========================================================================
init            gid             Numeric (``$1``) argument for the global identifier
                morphology_dir  String (``$o1``)
                morphology_name  
clear           void            Clears the model so there are no references, including circular ones
re_init_rng     channel_seed    Re-initialize the random; applies new random seed to all applicable (?)
=============== =============== ========================================================================

Properties:
-----------

=============== ============= ========================================================================
Property        Type          Description
=============== ============= ========================================================================
gid             float         A location where the simulator can save the GID assigned to the cell
all             SectionList   All the sections
APC             SectionList   APC
apical          SectionList   Apical sections
axonal          SectionList   Axonal sections
basal           SectionList   Basal sections
myelinated      SectionList   Myelinated sections
somatic         SectionList   Somatic sections
soma            section name  soma
dend            section name  dend
apic            section name  apic
axon            section name  axon
myelin          section name  myelin
nSecAll         float         Number of all the sections, before axon being reduced
nSecAxonalOrig  float         Number of sections in the axon, before being reduced
=============== ============= ========================================================================

Semantic order
--------------

`init` is called when loading a :ref:`Sonata Node file <node_file>` and instantiating a cell HOC object during the simulation. The emodel file name is defined by the node attribute `model_template`, and the morphology file name is defined by the node attribute `morphology`.
The paths to the folders containing the emodel HOC files and morphology files are specified in the :ref:`SONATA circuit config file <sonata_config>`. The parameter `biophysical_neuron_models_dir` defines the path for the emodel files, while `morphology_dir` and `alternate_morphologies` defines the path for the morphology files.

 * when should axon be shortened?

`re_init_rng` is called after instantiating a cell object to reinitialize the random number generator using the `ionchannel_seed` defined in the :ref:`SONATA simulation config file <sonata_simulation>`.

`clear` can be called when it is necessary to destroy the cell HOC object, for example, when the simulation is finished or when the cell object is removed from the simulation. It clears all references to the cell, including circular references that could prevent garbage collection.

.. _HOC: https://nrn.readthedocs.io/en/latest/hoc/index.html
.. _bluecellulab: https://bluecellulab.readthedocs.io/en/latest/
.. _neurodamus: https://neurodamus.readthedocs.io/en/stable/
