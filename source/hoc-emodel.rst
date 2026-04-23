HOC E-Model Templates
=====================

When instantiating a morphology in NEURON, the `HOC`_ language is used. For a model to be compatible with `BlueCelluLab`_ or `Neurodamus`_, it must implement a minimum set of public members. This document outlines the required attributes and functions.

Functions
---------

================ ================= ====================================================================
Name             Arguments         Description
================ ================= ====================================================================
init             gid               Numeric (``$1``) argument for the global identifier
                 morphology_dir    String (``$o1``) the path to morphology file directory
                 morphology_name   String (``$o2``) the name of the morphology file
clear            void              Clears all model references, including circular ones, for cleanup
re_init_rng      channel_seed      Re-initializes the random number generator with the specified seed
================ ================= ====================================================================

Properties
----------

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

Axon Replacement
----------------

The original axon from the loaded morphology is replaced with a shorter artificial axon stub (40-60 micrometers) mainly containing 1-2 axon sections representing axon initial segment (AIS)/ or axonal hillock. These initial sections form the origin site of action potentials during somatic current injections due high density of fast-activating sodium channels. These two sections are generally followed by myelinated axon ~1000 micrometers. Other sections of the original axon are removed for simulations to reduce the complexity of the model and save on computation time. Axons were considered neurites that mainly carry electrical signals away from the soma towards the post-synaptic neurons. However, simulating the whole axon is essential for studying extracellular potentials and several axonal studies suggest the increasing importance of whole axon simulations.

The `replace_axon()` function used for most BBP and OBI (Open Brain Institute) hoc templates can be found here in `BluePyEModel <https://github.com/openbraininstitute/BluePyEModel/blob/aea5174fcc8eecfba4333c402aebe7c00e26903e/bluepyemodel/evaluation/modifiers.py#L145>`_ and
`BluePyOpt <https://github.com/openbraininstitute/BluePyOpt/blob/21f0d68462cb918d4a2330b343ddd63c5569ec22/bluepyopt/ephys/morphologies.py#L183>`_. The diameters of the original axon morphology sections (if present) are used to create the new axon sections. 
The section IDs of the newly created initial axon sections and myelinated axon sections in BluePyEModel's `replace axon <https://github.com/openbraininstitute/BluePyEModel/blob/aea5174fcc8eecfba4333c402aebe7c00e26903e/bluepyemodel/evaluation/modifiers.py#L145>`_ function
are updated to the section IDs of the original axon morphology.

For example, by executing these statements in the `replace_axon()` function, before deleting the original axon:

.. code-block:: python

    axon[0] i1 = v(0.0001)
    axon[1] i2 = v(0.0001) 
    axon[2] i3 = v(0.0001)

After deleting the original axon and creating new axon & myelinated sections, the section IDs are updated as follows:

.. code-block:: python

    axon[0] v(0.0001) = i1
    axon[0] v(0.0001) = i2
    myelin v(0.0001) = i3

Hence, the new sections do not use new section ids. The section ids of the remaining axon sections from original morphology are not used during simulations.

Section ID
----------

Section ID is a unique integer identifier (starting from 0) for a section in the morphology. These are used to identify the sections in the SONATA node and edge files, used during SONATA report creation, and to specify section IDs in compartment_sets. The IDs start from 0 and are assigned based on the initialization in the NEURON simulator. If the original sections are deleted, the section IDs of the neuron should be updated by the HOC template through reinitialization. Similarly, when new sections are added, they should be initialized. If the model is not re-initialized, the section IDs of the new sections are assigned starting after the last ID of the original model.

The main `replace axon <https://github.com/openbraininstitute/BluePyEModel/blob/aea5174fcc8eecfba4333c402aebe7c00e26903e/bluepyemodel/evaluation/modifiers.py#L145>`_ function used in the BBP/OBI models adds 2 axon sections and a myelinated axon.

Template Versions
-----------------
The Blue Brain Project has used various versions of the HOC template over time due to changing scientific requirements and updates in the simulators: `Neurodamus`_ and `BlueCelluLab`_. The current version used in the OBI template is called `v6` for most single neuron models.

.. _HOC: https://nrn.readthedocs.io/en/latest/hoc/index.html
.. _BlueCelluLab: https://bluecellulab.readthedocs.io/en/latest/
.. _Neurodamus: https://neurodamus.readthedocs.io/en/stable/
