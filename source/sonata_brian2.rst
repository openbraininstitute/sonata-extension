Describing Brian 2 Models with SONATA
=====================================

Overview
--------

`Brian2 <https://brian2.readthedocs.io/en/stable/>`_ is a spiking neural network simulator.
This document describes how a network is defined using SONATA, to be loaded by the `Brian2` simulator.

.. warning::
   This is a preliminary draft, and does not handle several facets of circuits.
   It is also a living document; as features are added, please update them.
   See `Future Work` below for what needs to be added.

Physiology
----------

`Brian2` describes neuron physiology with a set of equations supplied at `NeuronGroup` creation time.

Ex:

.. code-block:: python

    NeuronGroup( # create neurons
        model='''dv/dt = (v_0 - v + g) / t_mbr : volt (unless refractory)
                dg/dt = -g / tau               : volt (unless refractory) 
                rfc                            : second
                ''',
                [...]
    )

This is translated into `JSON` under the `params` object, which is then used to parameterize the `NeuronGroup` call, including all options (like 'model', 'method', 'threshold', etc).
Within `params`, multiline values can be wrapped in a list to make it more readable.
The `Brian2` simulator is also particular about units, so they must be encoded when using the `namespace` keyword.
For this encoding to work, the following pattern is used: `'tau': [5, 'ms'],`; the loader will then apply the correct units at instantiation time.
Each different set of parameters and equations is stored in a .json file under the `components::biophysical_neuron_models_dir` as specified in `the SONATA extensions <https://sonata-extension.readthedocs.io/en/latest/sonata_config.html#components>`_.
The associated name is stored in the `model_template` nodes field.
Per neuron parameters are noted in the `dynamics_params` JSON object, as `name`, `unit` tuples.
These values are then loaded from the `nodes` file.

Thus, an example model template would be:

.. code-block:: json

   {
      "params": {
        "model": ["dv/dt = (v_0 - v + g) / t_mbr : volt (unless refractory)",
                  "dg/dt = -g / tau              : volt (unless refractory)",
                  "rfc                           : second"],
        "method": "linear",
        "threshold": "v > v_th",
        "reset": "v = v_rst; w = 0; g = 0 * mV",
        "refractory": "rfc"
      },
      "namespace": {
        "t_mbr": [20, "ms"],
        "tau": [5, "ms"],
        "v_0": [-52, "mV"],
        "v_th": [-45, "mV"],
        "v_rst": [-52, "mV"]
      },
      "dynamics_params": {
        "w_syn": "mV"
      }
    }



The edges (ie: synapses) are parameterized in the same way; there is a template per `synapse_type`.

Nodes
-----

The following are required for all models:

.. table::

    ================== =============================== ========== =========================================================================================
    Group              Field                           Type       Description
    ================== =============================== ========== =========================================================================================
    /0                 model_template                  utf8       Reference a template or class describing the electrophysical properties and mechanisms.
    /0                 model_type                      utf8       `biophysical`
    /0                 synapse_class                   utf8       Defines the synapse type of the node; whether the neuron is inhibitory or excitatory. "EXC" or "INH".
    /dynamics_params   ...                             float32    Values described in `dynamics_params`
    ================== =============================== ========== =========================================================================================

Values in the `model_template` `dynamics_params` section are read from the `dynamics_params`, and a per node value is registered in the simulator.

Edges
-----

.. table::

    ================== =============================== ========== =========================================================================================
    Group              Field                           Type       Description
    ================== =============================== ========== =========================================================================================
    /0                 synapse_type                    utf8       Model template from the `biophysical_neuron_models_dir`
    /dynamics_params   ...                             float32    Per synapse values (ex: `w`, `pre.delay`)
    ================== =============================== ========== =========================================================================================

Inputs
------

From James:

.. code-block:: json

    'inputs': {
        'poisson_sugar': {
            'input_type': 'spikes',
            'module': 'poisson',
            'node_set': 'sugar_neurons',
            'rate': POISSON_RATE,   # in Hz
            'weight': POISSON_W,
            'target_var': 'v',
            'zero_refractory': True,
        }
    }

Future Work
-----------

* The initial implementation was conceived to support the `A leaky integrate-and-fire computational model based on the connectome of the entire adult Drosophila brain reveals insights into sensorimotor processing. <https://github.com/philshiu/Drosophila_brain_model>`_
* The `inputs` block needs further refinement
* Only a single population is currently supported
