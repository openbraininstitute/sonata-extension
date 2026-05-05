Describing Brian 2 Models with SONATA
=====================================

Overview
========

`Brian2 <https://brian2.readthedocs.io/en/stable/>`_ is a spiking neural network simulator.
This document describes how a network is defined using SONATA, to be loaded by the `Brian2` simulator.

.. warning::
   This is a preliminary draft, and does not handle several facets of circuits.
   It is also a living document; as features are added, please update them.
   See `Future Work` below for what needs to be added.

Physiology
----------

`Brian2` describes neuron physiology with a set of equations supplied at `NeuronGroup <https://brian2.readthedocs.io/en/2.5.3/reference/brian2.groups.neurongroup.NeuronGroup.html>`_ creation time.

Ex:

.. code-block:: python

    NeuronGroup( # create neurons
        model='''dv/dt = (v_0 - v + g) / t_mbr : volt (unless refractory)
                dg/dt = -g / tau               : volt (unless refractory)
                rfc                            : second
                ''',
        method,
        method_options,
        threshold,
        reset,
        refractory,
                [...]
    )

Thus, for the `model_template` arguments, one can save have keys within the `params` of `model`, `method`, `method_options`, `threshold`, `reset`, `refractory`.
Within `params`, multiline values can be wrapped in a list to make it more readable.
The `Brian2` simulator is also particular about units, so they must be encoded when using the `namespace` keyword.
For this encoding to work, the following pattern is used: `'tau': [5, 'ms'],`; the loader will then apply the correct units at instantiation time.
Each different set of parameters and equations is stored in a .json file under the `components::point_neuron_models_dir` as specified in `the SONATA extensions <https://sonata-extension.readthedocs.io/en/latest/sonata_config.html#components>`_.
The associated name is stored in the `model_template` nodes field.
Per neuron parameters are noted in the `dynamics_params` `JSON` object, as `name`, `unit` keys.
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
        "v_rst": [-52, "mV"]
      },
      "dynamics_params": {
        "v_th": "mV"
      }
    }



The edges (ie: synapses) are parameterized in the same way; there is a template per `synapse_type`.
In this example, there is a `dynamics_params/v_th` attribute in the nodes.h5 file, and when the simulator reads the module template, it will use that data to parameterize the neurons.

Nodes
-----

The following are required for all models - it has the `type` of `brian2_point` for purposes of the SONATA circuit config file.

.. table::

    ================== =============================== ========== =========================================================================================
    Group              Field                           Type       Description
    ================== =============================== ========== =========================================================================================
    /0                 model_template                  utf8       Reference a template or class describing the electrophysical properties and mechanisms.
    /0                 model_type                      utf8       `brian2_point`
    /0                 synapse_class                   utf8       Defines the synapse type of the node; whether the neuron is inhibitory or excitatory. "EXC" or "INH".
    /0/dynamics_params ...                             float32    Values described in `dynamics_params`
    ================== =============================== ========== =========================================================================================

Values in the `model_template` `dynamics_params` section are read from the `dynamics_params`, and a per node value is registered in the simulator.
For the example model template above, there would be `/0/dynamics_params/v_th`.

Edges
-----

The following are required for all models - it has the `type` of `brian2_synapse` for purposes of the SONATA circuit config file.

.. table::

    ================== =============================== ========== =========================================================================================
    Group              Field                           Type       Description
    ================== =============================== ========== =========================================================================================
    /0                 synapse_type                    utf8       Model template from the `point_neuron_models_dir`
    /dynamics_params   ...                             float32    Per synapse values (ex: `w`, `pre.delay`)
    ================== =============================== ========== =========================================================================================

Inputs
------

Poisson Spike
*************

A new `input_type`::`module` is defined for Poisson Spike:

.. code-block:: json

    "inputs": {
        "poisson_sugar": {
            "input_type": "spikes",
            "module": "poisson",
            "node_set": "sugar_neurons",
            "rate": "POISSON_RATE_in_Hz",
            "weight": "POISSON_W",
        }
    }

Examples
========

FlyWire
-------

Physiology
**********

For the model template, this is used:

.. code-block:: json

    {
      "params": {
        "model": [
          "dv/dt = (v_0 - v + g) / t_mbr : volt (unless refractory)",
          "dg/dt = -g / tau               : volt (unless refractory)",
          "rfc                            : second",
        ],
        "method": "linear",
        "threshold": "v > v_th",
        "reset": "v = v_rst; w = 0; g = 0 * mV",
        "refractory": "rfc"
      },
      "namespace": {
        "t_mbr": [20.0, "ms"],
        "tau": [5.0, "ms"],
        "v_0": [-52.0, "mV"],
        "v_th": [-45.0, "mV"],
        "v_rst": [-52.0, "mV"]
      },
      "initial": {
        "v": [-52.0, "mV"],
        "g": [0, "mV"],
        "rfc": [2.2, "ms"]
      }
    }


For the synapse template, this is used:

.. code-block:: json

    {
      "params": {
        "model": "w : volt",
        "on_pre": "g += w",
        "delay": [1.8, "ms"]
      },
      "dynamics": {
        "w": "mV"
      }
    }

Nodes
*****

The nodes files doesn't contain much information per node, since most of the values are constant and set by the model template.

.. code-block::

    /nodes/drosophila/0/@library/model_template Dataset {1}
    /nodes/drosophila/0/@library/model_type Dataset {1}
    /nodes/drosophila/0/flywire_id Dataset {127400}
    /nodes/drosophila/0/model_template Dataset {127400}
    /nodes/drosophila/0/model_type Dataset {127400}
    /nodes/drosophila/node_type_id Dataset {127400}


Edges
*****

Only the `w` (or weight) is different per synapses, so it is stored along with the information about which `model_template` to use.

.. code-block::

    /edges/drosophila__drosophila__brian2_synapse/0/@library/model_template Dataset {1}
    /edges/drosophila__drosophila__brian2_synapse/0/model_template Dataset {14687178}
    /edges/drosophila__drosophila__brian2_synapse/0/w Dataset {14687178}
    /edges/drosophila__drosophila__brian2_synapse/edge_type_id Dataset {14687178}
    /edges/drosophila__drosophila__brian2_synapse/source_node_id Dataset {14687178}
    /edges/drosophila__drosophila__brian2_synapse/target_node_id Dataset {14687178}


Circuit Config
**************

.. code-block:: json

    {
      "components": {
        "point_neuron_models_dir": "models"
      },
      "node_sets_file": "node_sets.json",
      "target_simulator": "Brian2",
      "networks": {
        "nodes": [
          {
            "nodes_file": "nodes.h5",
            "populations": {
              "drosophila": {
                "type": "brian2_point"
              }
            }
          }
        ],
        "edges": [
          {
            "edges_file": "edges.h5",
            "populations": {
              "drosophila__drosophila__brian2_synapse": {
                "type": "brian2_synapse"
              }
            }
          }
        ]
      }
    }


Simulation Config
*****************

.. code-block:: json

    {
      "run": {
        "tstop": 1000,
        "dt": 0.1,
        "random_seed": 42
      },
      "target_simulator": "Brian2",
      "network": "../output/circuit_config.json",
      "inputs": {
          "poisson": {
              "input_type": "spikes",
              "module": "poisson",
              "node_set": "sugar",
              "delay": 0,
              "duration": 1000,
              "rate": 150,
              "weight": 68.75
          }
      }
    }



Future Work
-----------

* The initial implementation was conceived to support the `A leaky integrate-and-fire computational model based on the connectome of the entire adult Drosophila brain reveals insights into sensorimotor processing. <https://github.com/philshiu/Drosophila_brain_model>`_
* The `inputs` block needs further refinement
* Only a single population is currently supported
