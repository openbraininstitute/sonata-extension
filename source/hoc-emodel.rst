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
nSecAxonalOrig  float         Number of sections in the axon, before being reduced
=============== ============= ========================================================================

Semantic order
--------------

Describe what is called when.

Ex:
 * When `init` is called
 * When `clear` is called
 * when should axon be shortened?
 * when is `re_init_rng` called? Under which circumstances.

.. _HOC: https://nrn.readthedocs.io/en/latest/hoc/index.html
.. _bluecellulab: https://bluecellulab.readthedocs.io/en/latest/
.. _neurodamus: https://neurodamus.readthedocs.io/en/stable/
