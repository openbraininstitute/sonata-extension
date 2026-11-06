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

Populations
-----------

A `Brian2` circuit has one or more node populations of `type` `brian2_point`, and any number of node populations of `type` `virtual` (i.e., the source nodes of projections, see :ref:`virtual population <virtual_node_type>`).

* Each `brian2_point` population becomes one `NeuronGroup <https://brian2.readthedocs.io/en/stable/reference/brian2.groups.neurongroup.NeuronGroup.html>`_.
  All its nodes use the same neuron model template.
* A `virtual` population has no dynamics. It emits spikes only when a :ref:`synapse_replay <spikes-synapse-replay>` input names it (see `Inputs`); otherwise it is not instantiated.
* Edge populations have the `type` `brian2_synapse`.
  An edge population connects any node population to a `brian2_point` population; edges into a `virtual` population are not allowed.
  Each edge population becomes one `Synapses <https://brian2.readthedocs.io/en/stable/reference/brian2.synapses.synapses.Synapses.html>`_ object, and all its edges use the same synapse model template.

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
        method,
        threshold,
        reset,
        refractory,
                [...]
    )

Neuron model templates
**********************

Each different set of parameters and equations is stored in a `.json` file under the population's `point_neuron_models_dir`, as specified in :ref:`components` (set once under `components`, or per population).
A node's `model_template` field names it as `json:<name>`, which resolves to `<point_neuron_models_dir>/<name>.json`.

A neuron model template has these keys:

.. table::

    ================== ============ =========================================================================================
    Key                Requirement  Description
    ================== ============ =========================================================================================
    params             Mandatory    `model`, `method`, `threshold`, `reset` and `refractory`, passed to `NeuronGroup`.
                                    Multiline values can be wrapped in a list to make them more readable.
    namespace          Optional     Constants shared by all nodes of the population, as `name: [value, unit]`.
    dynamics_params    Optional     Per-node parameters, as `name: unit`; the values are read from `/0/dynamics_params/<name>` in the nodes file.
    initial            Optional     Initial values of state variables, as `name: [value, unit]`.
    ================== ============ =========================================================================================

The `Brian2` simulator is particular about units, so they must be encoded.
For a value, the pattern is `'tau': [5, 'ms']`; for a per-node parameter, it is the unit of the values in the nodes file, e.g. `'v_th': 'mV'`.
Units are names from `brian2.units`, or `"1"` for dimensionless values.

* A per-node parameter must be declared in `model` as a constant of its dimension, e.g. `v_th : volt (constant)`, and must not also appear in `namespace`.
* `model` must declare `I_inj : amp`: the loader replaces it with the sum of the current injections of the simulation config's `inputs`.
* `initial` values are set after the `NeuronGroup` is created; the simulation config's `conditions` `v_init` then sets `v` of every node.

Thus, an example model template would be:

.. code-block:: json

   {
      "params": {
        "model": ["dv/dt = (v_0 - v + g + I_inj * R_m) / t_mbr : volt (unless refractory)",
                  "dg/dt = -g / tau                          : volt (unless refractory)",
                  "rfc                                       : second",
                  "v_th                                      : volt (constant)",
                  "I_inj                                     : amp"],
        "method": "linear",
        "threshold": "v > v_th",
        "reset": "v = v_rst; g = 0 * mV",
        "refractory": "rfc"
      },
      "namespace": {
        "t_mbr": [20, "ms"],
        "tau": [5, "ms"],
        "R_m": [10, "Mohm"],
        "v_0": [-52, "mV"],
        "v_rst": [-52, "mV"]
      },
      "dynamics_params": {
        "v_th": "mV"
      },
      "initial": {
        "rfc": [2.2, "ms"]
      }
    }

In this example, the nodes file has a `/0/dynamics_params/v_th` dataset, and each node gets its own threshold from it.

Synapse model templates
***********************

The edges (ie: synapses) are parameterized in the same way, with a template per edge population, located under the edge population's `point_neuron_models_dir`.
A synapse model template has these keys:

.. table::

    ================== ============ =========================================================================================
    Key                Requirement  Description
    ================== ============ =========================================================================================
    params             Mandatory    `model` and `on_pre`, passed to `Synapses`, and `delay` as `[value, unit]`, the delay of every synapse.
    dynamics           Optional     Per-edge values, as `name: unit`; the values are read from `/0/<name>` in the edges file, e.g. the weight `w`.
    ================== ============ =========================================================================================

Listing `delay` in `dynamics` (e.g. `"delay": "ms"`) gives each synapse the axonal delay stored in the edges file's standard `/0/delay` dataset, instead of the `params` `delay`.

.. code-block:: json

    {
      "params": {
        "model": "w : volt",
        "on_pre": "g += w",
        "delay": [1.8, "ms"]
      },
      "dynamics": {
        "w": "mV",
        "delay": "ms"
      }
    }

Nodes
-----

The following are required for all `brian2_point` populations - they have the `type` of `brian2_point` for purposes of the SONATA circuit config file.

.. table::

    ================== =============================== ========== =========================================================================================
    Group              Field                           Type       Description
    ================== =============================== ========== =========================================================================================
    /0                 model_template                  utf8       The neuron model template, as `json:<name>`; the same for all nodes of the population (an `@library` enumeration is recommended).
    /0                 model_type                      utf8       `brian2_point`
    /0                 synapse_class                   utf8       Defines the synapse type of the node; whether the neuron is inhibitory or excitatory. "EXC" or "INH".
    /0/dynamics_params ...                             float32    Per-node values of the template's `dynamics_params`
    /                  node_type_id                    int        Set to -1.
    ================== =============================== ========== =========================================================================================

Values in the `model_template` `dynamics_params` section are read from the `dynamics_params`, and a per node value is registered in the simulator.
For the example model template above, there would be `/0/dynamics_params/v_th`.

`virtual` populations follow the :ref:`virtual population <virtual_node_type>` fields of the technical description; they need no model template.

Edges
-----

The following are required for all models - it has the `type` of `brian2_synapse` for purposes of the SONATA circuit config file.

.. table::

    ================== =============================== ========== =========================================================================================
    Group              Field                           Type       Description
    ================== =============================== ========== =========================================================================================
    /0                 model_template                  utf8       The synapse model template, as `json:<name>`; the same for all edges of the population (an `@library` enumeration is recommended).
    /0                 ...                             float32    Per-edge values of the template's `dynamics`, e.g. `w`.
    /0                 delay                           float32    Optional: the axonal delay in ms, used when the template lists `delay` in `dynamics`.
    /                  edge_type_id                    int        Set to -1.
    /                  source_node_id                  uint       The id of the presynaptic node.
    /                  target_node_id                  uint       The id of the postsynaptic node.
    /indices           ...                                        The SONATA edge indices (e.g. written with `libsonata.EdgePopulation.write_indices`).
    ================== =============================== ========== =========================================================================================

`source_node_id` and `target_node_id` datasets have an HDF5 attribute of type string named `node_population` defining the source and target node population name respectively.

An edge population holds at most one edge per (source, target) pair.
A circuit converted from one with several synapses per connection merges them into one edge; `sonata-simplify` sums their signed weights and uses the weight-weighted mean of their delays.

Inputs
------

The simulation config's `inputs` are applied as follows; the `node_set` of an input selects the nodes it acts on, in any `brian2_point` population.

.. table::

    ============================== ============== =========================================================================================
    `module`                       `input_type`   Description
    ============================== ============== =========================================================================================
    `linear`, `pulse`              current_clamp  Current injected through `I_inj`. Inputs with the same `node_set` are summed.
    `sinusoidal`                   current_clamp  As above; its `dt` must equal the simulation's `dt`.
    `poisson`                      spikes         Independent Poisson spike trains kicking the membrane potential directly; see below.
    `synapse_replay`               spikes         Spikes of the file's population delivered through that population's edges; see below.
    ============================== ============== =========================================================================================

Other modules, and `compartment_set`, are not supported.

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
            "weight": "POISSON_W"
        }
    }

Each node of `node_set` receives its own Poisson train of `rate` (Hz) between `delay` and `delay + duration`; each spike adds `weight` (mV) to its `v`.

Synapse replay
**************

The :ref:`synapse_replay <spikes-synapse-replay>` `spike_file` is a SONATA spike file with `/spikes/<population>/node_ids` and `/spikes/<population>/timestamps` for each population it covers.
The spikes of each population are delivered through every edge population whose source is that population, to the targets in `node_set`.
A population can be

* a `virtual` population: the projection is driven, as for `NEURON` circuits; or
* a `brian2_point` population: its spikes are replayed through its own edges, on top of the simulated activity.

Spikes later than `duration` are dropped, and `delay` is added to the times of the others.
`Brian2` allows one spike per source and time step, so a second spike of the same source within one `dt` is dropped.

.. code-block:: json

    "inputs": {
        "vpm_spikes": {
            "input_type": "spikes",
            "module": "synapse_replay",
            "spike_file": "vpm_spikes.h5",
            "node_set": "All",
            "delay": 0,
            "duration": 1000
        }
    }

Outputs
-------

The spikes of every `brian2_point` population are written to the simulation config's `spikes_file`, one group per population.
Reports are soma reports of `v` (`type` `compartment`, `sections` `soma`, `variable_name` `v`) with the simulation's `dt`, written with one group per population.

Connection Overrides
--------------------

The simulation config's :ref:`connection_overrides <connection_overrides>` act on the synapses that connect `source` to `target`: of every edge population between their populations, including the synapses built for :ref:`synapse_replay <spikes-synapse-replay>` inputs.
`source` may select `virtual` nodes.
`Brian2` supports these properties:

.. table::

    ======================= =========================================================================================
    Property                Description
    ======================= =========================================================================================
    name                    As in SONATA.
    source                  As in SONATA.
    target                  As in SONATA.
    weight                  Factor on each synapse's weight `w` as loaded from the edges file; see below.
    synapse_delay_override  Replaces each synapse's delay, in ms.
    delay                   Time in ms at which the override takes effect; `0`, the default, applies it from the start.
    ======================= =========================================================================================

`weight` is always a factor on the weight the circuit gives each synapse, never on the synapse's current value, so overrides do not compound.
A `weight` of `0` disconnects, a later `weight` of `1` restores the circuit's weights exactly, inhibitory (negative) weights included, and a `weight` of `0.5` halves them however many overrides came before.
This is the SONATA definition of `weight` as a multiplier of the weight stored in the edges file, applied to the synapse model's `w`.

`synapse_delay_override` is applied at `delay` too, rather than being ignored when `delay` is non-zero.
`spont_minis`, `synapse_configure`, `modoverride`, `neuromodulation_dtc` and `neuromodulation_strength` have no `Brian2` counterpart, and setting any of them is an error.

Ex: disconnecting the sugar neurons from the rest of the circuit between 200 and 400 ms:

.. code-block:: json

    "connection_overrides": [
        {"name": "disconnect_sugar", "source": "sugar", "target": "All", "delay": 200, "weight": 0.0},
        {"name": "reconnect_sugar", "source": "sugar", "target": "All", "delay": 400, "weight": 1.0}
    ]

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


Simplified cortical circuit
---------------------------

A circuit exported by `sonata-simplify` (`adex_brian2`) from an extracted cortical circuit: one `brian2_point` population of AdEx neurons with per-neuron parameters, and its thalamic `virtual` input population `VPM`.

Physiology
**********

For the neuron model template (`models/adex.json`), the parameters fitted per neuron are `dynamics_params`; those shared by all neurons are in `namespace`:

.. code-block:: json

    {
      "params": {
        "model": [
          "dv/dt = (g_L * (E_L - v) + g_L * Delta_T * exp((v - V_th) / Delta_T) - w_adapt + I_syn + I_inj) / C_m : volt (unless refractory)",
          "dw_adapt/dt = (a * (v - E_L) - w_adapt) / tau_w : amp",
          "dg_ex/dt = -g_ex / tau_syn_ex : siemens",
          "dg_in/dt = -g_in / tau_syn_in : siemens",
          "I_syn = g_ex * (E_ex - v) + g_in * (E_in - v) : amp",
          "I_inj : amp",
          "refractory_period : second",
          "C_m : farad (constant)",
          "g_L : siemens (constant)",
          "E_L : volt (constant)",
          "V_th : volt (constant)",
          "V_reset : volt (constant)",
          "a : siemens (constant)",
          "b : amp (constant)",
          "tau_w : second (constant)"
        ],
        "method": "euler",
        "threshold": "v > V_peak",
        "reset": "v = V_reset; w_adapt += b",
        "refractory": "refractory_period"
      },
      "namespace": {
        "Delta_T": [2.0, "mV"],
        "V_peak": [0.0, "mV"],
        "E_ex": [0.0, "mV"],
        "E_in": [-80.0, "mV"],
        "tau_syn_ex": [1.64, "ms"],
        "tau_syn_in": [9.61, "ms"]
      },
      "dynamics_params": {
        "C_m": "pF",
        "g_L": "nS",
        "E_L": "mV",
        "V_th": "mV",
        "V_reset": "mV",
        "a": "nS",
        "b": "pA",
        "tau_w": "ms"
      },
      "initial": {
        "w_adapt": [0.0, "pA"],
        "g_ex": [0.0, "nS"],
        "g_in": [0.0, "nS"],
        "refractory_period": [2.0, "ms"]
      }
    }

For the synapse template (`models/synapse.json`), used by both edge populations, the sign of `w` encodes the polarity: positive weights are excitatory, negative ones inhibitory.
Each synapse has its own delay:

.. code-block:: json

    {
      "params": {
        "model": "w : siemens",
        "on_pre": "g_ex_post += (w > 0*nS) * w\ng_in_post += (w < 0*nS) * (-w)",
        "delay": [0.67, "ms"]
      },
      "dynamics": {
        "w": "nS",
        "delay": "ms"
      }
    }

Nodes
*****

`n` is the number of neurons and `m` the number of `VPM` fibres.

.. code-block::

    /nodes/S1nonbarrel_neurons/0/@library/model_template Dataset {1}
    /nodes/S1nonbarrel_neurons/0/model_template Dataset {n}
    /nodes/S1nonbarrel_neurons/0/model_type Dataset {n}
    /nodes/S1nonbarrel_neurons/0/synapse_class Dataset {n}
    /nodes/S1nonbarrel_neurons/0/dynamics_params/C_m Dataset {n}
    /nodes/S1nonbarrel_neurons/0/dynamics_params/g_L Dataset {n}
    [... one dataset per entry of the template's dynamics_params ...]
    /nodes/S1nonbarrel_neurons/node_type_id Dataset {n}
    /nodes/VPM/0/model_type Dataset {m}
    /nodes/VPM/node_type_id Dataset {m}

Edges
*****

One edge per (source, target) pair: `k` recurrent connections and `l` thalamic ones.

.. code-block::

    /edges/S1nonbarrel_neurons__S1nonbarrel_neurons__chemical/0/@library/model_template Dataset {1}
    /edges/S1nonbarrel_neurons__S1nonbarrel_neurons__chemical/0/model_template Dataset {k}
    /edges/S1nonbarrel_neurons__S1nonbarrel_neurons__chemical/0/w Dataset {k}
    /edges/S1nonbarrel_neurons__S1nonbarrel_neurons__chemical/0/delay Dataset {k}
    /edges/S1nonbarrel_neurons__S1nonbarrel_neurons__chemical/edge_type_id Dataset {k}
    /edges/S1nonbarrel_neurons__S1nonbarrel_neurons__chemical/source_node_id Dataset {k}
    /edges/S1nonbarrel_neurons__S1nonbarrel_neurons__chemical/target_node_id Dataset {k}
    /edges/S1nonbarrel_neurons__S1nonbarrel_neurons__chemical/indices [...]
    /edges/VPM__S1nonbarrel_neurons__chemical/0/@library/model_template Dataset {1}
    /edges/VPM__S1nonbarrel_neurons__chemical/0/model_template Dataset {l}
    /edges/VPM__S1nonbarrel_neurons__chemical/0/w Dataset {l}
    /edges/VPM__S1nonbarrel_neurons__chemical/0/delay Dataset {l}
    /edges/VPM__S1nonbarrel_neurons__chemical/edge_type_id Dataset {l}
    /edges/VPM__S1nonbarrel_neurons__chemical/source_node_id Dataset {l}
    /edges/VPM__S1nonbarrel_neurons__chemical/target_node_id Dataset {l}
    /edges/VPM__S1nonbarrel_neurons__chemical/indices [...]

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
              "S1nonbarrel_neurons": {"type": "brian2_point"},
              "VPM": {"type": "virtual"}
            }
          }
        ],
        "edges": [
          {
            "edges_file": "edges.h5",
            "populations": {
              "S1nonbarrel_neurons__S1nonbarrel_neurons__chemical": {"type": "brian2_synapse"},
              "VPM__S1nonbarrel_neurons__chemical": {"type": "brian2_synapse"}
            }
          }
        ]
      }
    }

Simulation Config
*****************

Thalamic spikes replayed onto all neurons, with the node sets `All` (`S1nonbarrel_neurons`) and `VPM`:

.. code-block:: json

    {
      "run": {
        "tstop": 1000,
        "dt": 0.1,
        "random_seed": 42
      },
      "target_simulator": "Brian2",
      "network": "circuit_config.json",
      "conditions": {
        "v_init": -70.0
      },
      "inputs": {
        "vpm_spikes": {
          "input_type": "spikes",
          "module": "synapse_replay",
          "spike_file": "vpm_spikes.h5",
          "node_set": "All",
          "delay": 0,
          "duration": 1000
        }
      },
      "output": {
        "output_dir": "output",
        "spikes_file": "spikes.h5"
      }
    }

Future Work
-----------

* The initial implementation was conceived to support the `A leaky integrate-and-fire computational model based on the connectome of the entire adult Drosophila brain reveals insights into sensorimotor processing. <https://github.com/philshiu/Drosophila_brain_model>`_
* The `inputs` block needs further refinement: relative and noise current clamps are not supported.
* One neuron model template per `brian2_point` population and one synapse model template per edge population.
* One edge per (source, target) pair in an edge population.
* Short-term plasticity, receptor kinetics, the extracellular calcium scaling of `u_hill_coefficient` and spontaneous release are not described yet.
