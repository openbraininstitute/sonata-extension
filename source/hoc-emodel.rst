HOC E-Model Templates
=====================

When instantiating a morphology in NEURON, the `HOC`_ language is used.
For a model to be compatible with `BlueCelluLab`_ or `Neurodamus`_, it must implement a minimum set of public members.
This document outlines the required attributes and functions.

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
all             SectionList   All the sections
apical          SectionList   Apical dendrite sections with NEURON section name: ``apic``
axonal          SectionList   Axonal sections with NEURON section name: ``axon``
basal           SectionList   Basal dendrite sections with NEURON section name: ``dend``
myelinated      SectionList   Myelinated sections with NEURON section name: ``myelin``
somatic         SectionList   Somatic sections with NEURON section name: ``soma``
soma            section name  soma
dend            section name  dend
apic            section name  apic
axon            section name  axon
myelin          section name  myelin
nSecAll         float         Number of all the sections, before axon being reduced
nSecAxonalOrig  float         Number of sections in the axon, before being reduced
=============== ============= ========================================================================

Note: The axon from original morphology is replaced with a shorter artificial axon stub during the axon replacement process to reduce the computational cost of simulating the axon. A myelinated section is then added after the artificial axon stub to act as a passive cable after the replaced axon. The myelinated section prevents any signal reflections from the end of the axon back to the soma as the potential can propagate from the replaced axon to the myelinated section. See :ref:`axon_replacement` for more details about the axon replacement process. 

Semantic order
--------------

`init` is called when loading a :ref:`Sonata Node file <node_file>` and instantiating a cell HOC object during the simulation.
The emodel file name is defined by the node attribute `model_template`, and the morphology file name is defined by the node attribute `morphology`.
The paths to the folders containing the emodel HOC files and morphology files are specified in the :ref:`SONATA circuit config file <sonata_config>`.
The parameter `biophysical_neuron_models_dir` defines the path for the emodel files, while `morphology_dir` and `alternate_morphologies` defines the path for the morphology files.

`re_init_rng` is called after instantiating a cell object to reinitialize the random number generator using the `ionchannel_seed` defined in the :ref:`SONATA simulation config file <sonata_simulation>`.

`clear` can be called when it is necessary to destroy the cell HOC object, for example, when the simulation is finished or when the cell object is removed from the simulation.
It clears all references to the cell, including circular references that could prevent garbage collection.

.. _axon_replacement:

Axon Replacement
----------------

The original axon from the loaded morphology is replaced with a shorter artificial axon stub (40-60 micrometers) mainly containing 1-2 axon sections representing axon initial segment (AIS)/ or axonal hillock.
These initial sections form the origin site of action potentials during somatic current injections due high density of fast-activating sodium channels.
These two sections are generally followed by myelinated axon ~1000 micrometers.
Other sections of the original axon are removed for simulations to reduce the complexity of the model and save on computation time.
Axons were considered neurites that mainly carry electrical signals away from the soma towards the post-synaptic neurons.
However, simulating the whole axon is essential for studying extracellular potentials and several axonal studies suggest the increasing importance of whole axon simulations.

The `replace_axon()` function used for most BBP and OBI (Open Brain Institute) hoc templates can be found here in `BluePyEModel <https://github.com/openbraininstitute/BluePyEModel/blob/0aeec3d5f9ce9087e31d8220478337d527edc586/bluepyemodel/evaluation/modifiers.py#L278>`_ (`Python function: <https://github.com/openbraininstitute/BluePyEModel/blob/aea5174fcc8eecfba4333c402aebe7c00e26903e/bluepyemodel/evaluation/modifiers.py#L145>`_). Some templates generated by BluePyOpt have used a different replace axon function: 
`BluePyOpt <https://github.com/openbraininstitute/BluePyOpt/blob/5cf752a5f630005b8b0f5353c882e73e154e94be/bluepyopt/ephys/morphologies.py#L226>`_ (`Python function: <https://github.com/openbraininstitute/BluePyOpt/blob/21f0d68462cb918d4a2330b343ddd63c5569ec22/bluepyopt/ephys/morphologies.py#L183>`_).

BBP/OBI Default (BluePyEModel) `replace_axon` function
This is the  `replace axon`_ function used in the BBP/OBI models. It deletes the original axon and adds 2 artificial axon sections and a myelinated axon. The axonal section length is 60 micrometers and have diameters of axon from original morphology when present. The myelinated axon length is 1000 micrometers and has a uniform diameter from one of the original axon sections. 
Note: This replace axon function requires that the original morphology has at least 3 axon sections.

See the code below for details:

.. code-block::

    proc replace_axon(){ local nSec, L_chunk, dist, i1, i2, count, L_target, chunkSize, L_real localobj diams, lens

        L_target = 60  // length of stub axon
        nseg0 = 5  // number of segments for each of the two axon sections

        nseg_total = nseg0 * 2
        chunkSize = L_target/nseg_total

        nSec = 0
        forsec axonal{nSec = nSec + 1}

        // Try to grab info from original axon
        if(nSec < 3){ //At least two axon sections have to be present!

            execerror("Less than three axon sections are present! This emodel can't be run with such a morphology!")

        } else {

            diams = new Vector()
            lens = new Vector()

            access axon[0]
            axon[0] i1 = v(0.0001) // used when serializing sections prior to sim start
            axon[1] i2 = v(0.0001) // used when serializing sections prior to sim start
            axon[2] i3 = v(0.0001) // used when serializing sections prior to sim start

            count = 0
            forsec axonal{ // loop through all axon sections

                nseg = 1 + int(L/chunkSize/2.)*2  //nseg to get diameter

                for (x) {
                    if (x > 0 && x < 1) {
                        count = count + 1
                        diams.resize(count)
                        diams.x[count-1] = diam(x)
                        lens.resize(count)
                        lens.x[count-1] = L/nseg
                        if( count == nseg_total ){
                            break
                        }
                    }
                }
                if( count == nseg_total ){
                    break
                }
            }

            // get rid of the old axon
            forsec axonal{delete_section()}
            execute1("create axon[2]", CellRef)

            L_real = 0
            count = 0

            // new axon dependant on old diameters
            for i=0,1{
                access axon[i]
                L =  L_target/2
                nseg = nseg_total/2

                for (x) {
                    if (x > 0 && x < 1) {
                        diam(x) = diams.x[count]
                        L_real = L_real+lens.x[count]
                        count = count + 1
                    }
                }

                all.append()
                axonal.append()

                if (i == 0) {
                    v(0.0001) = i1
                } else {
                    v(0.0001) = i2
                }
            }

            nSecAxonal = 2
            soma[0] connect axon[0](0), 1
            axon[0] connect axon[1](0), 1

            create myelin[1]
            access myelin{
                    L = 1000
                    diam = diams.x[count-1]
                    nseg = 5
                    v(0.0001) = i3
                    all.append()
                    myelinated.append()
            }
            connect myelin(0), axon[1](1)
        }
    }    

BluePyOpt `replace_axon` function

This is the `replace axon`_ function used in some BluePyOpt templates. It deletes the original axon and adds 2 artificial axon sections and a myelinated axon. The axonal section length is 60 micrometers and have diameters of axon from original morphology when present. The myelinated axon length is 1000 micrometers and has a uniform diameter from one of the original axon sections. See the code below for details:
Note: This replace axon function can work for original neuron morphologies with no or some axon sections.


.. code-block::

    proc replace_axon(){ local nSec, D1, D2
    // preserve the number of original axonal sections
    nSec = sec_count(axonal)

    // Try to grab info from original axon
    if(nSec == 0) { //No axon section present
        D1 = D2 = 1
    } else if(nSec == 1) {
        axon[0] D1 = D2 = diam
    } else {
        axon[0] D1 = D2 = diam
        soma distance() //to calculate distance from soma
        forsec axonal{
        //if section is longer than 60um then store diam and exit from loop
        if(distance(0.5) > 60){
            D2 = diam
            break
        }
        }
    }

    // get rid of the old axon
    forsec axonal{
        delete_section()
    }

    create axon[2]

    axon[0] {
        L = 30
        diam = D1
        nseg = 1 + 2*int(L/40)
        all.append()
        axonal.append()
    }
    axon[1] {
        L = 30
        diam = D2
        nseg = 1 + 2*int(L/40)
        all.append()
        axonal.append()
    }
    nSecAxonal = 2
    soma[0] connect axon[0](0), 1
    axon[0] connect axon[1](0), 1
    }

The diameters of the original axon morphology sections (if present) are used to create the new axon sections.
The section IDs of the newly created initial axon sections and myelinated axon sections in BluePyEModel's `replace axon <https://github.com/openbraininstitute/BluePyEModel/blob/aea5174fcc8eecfba4333c402aebe7c00e26903e/bluepyemodel/evaluation/modifiers.py#L145>`_ function are updated to the section IDs of the original axon morphology.

For example, by executing these statements in the `replace_axon()` function, before deleting the original axon:

.. code-block::

    axon[0] i1 = v(0.0001)
    axon[1] i2 = v(0.0001)
    axon[2] i3 = v(0.0001)

After deleting the original axon and creating new axon & myelinated sections, the section IDs are updated as follows:

.. code-block::

    axon[0] v(0.0001) = i1
    axon[0] v(0.0001) = i2
    myelin v(0.0001) = i3

Hence, the new sections do not use new section ids. The section ids of the remaining axon sections from original morphology are not used during simulations.

Section ID
----------

Section ID is a unique integer identifier (starting from 0) for a section in the morphology.
These are used to identify the sections in the SONATA node and edge files, used during SONATA report creation, and to specify section IDs in compartment_sets.
The IDs start from 0 and are assigned based on the initialization in the NEURON simulator.
If the original sections are modified e.g. by `replace_axon` function or by adding/deleting sections, the section IDs of the neuron should be updated by the HOC template through reinitialization.
Similarly, when new sections are added, they should be initialized.
If the model is not re-initialized, the section IDs of the new sections are assigned starting after the last ID of the original model.


Template Versions
-----------------
The Blue Brain Project has used various versions of the HOC template over time due to changing scientific requirements and updates in the simulators: `Neurodamus`_ and `BlueCelluLab`_. The current version used in the OBI template is called `v6` for most single neuron models.
The current version (May 2026) of the hoc template is available at `here <https://github.com/openbraininstitute/BluePyEModel/blob/main/bluepyemodel/export_emodel/templates/cell_template_neurodamus_sbo.jinja2>`_ in the `BluePyEModel`_ repository.

.. _HOC: https://nrn.readthedocs.io/en/latest/hoc/index.html
.. _BlueCelluLab: https://bluecellulab.readthedocs.io/en/latest/
.. _Neurodamus: https://neurodamus.readthedocs.io/en/stable/
.. _BluePyEModel: https://github.com/openbraininstitute/BluePyEModel
.. _BluePyOpt: https://github.com/openbraininstitute/BluePyOpt
