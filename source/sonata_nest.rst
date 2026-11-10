.. _sonata_nest:

Describing NEST Models with SONATA
==================================

Overview
--------

`NEST <https://nest-simulator.readthedocs.io/>`_ is a simulator for networks of point neurons. This document describes how such a network is encoded in SONATA so it can be read by `libsonata` and instantiated in NEST through the PyNEST ``Create``/``Connect`` API.

A NEST circuit uses ``target_simulator: "NEST"`` and is described by:

- the circuit config: each population's ``type`` and ``point_neuron_models_dir``;
- the nodes/edges files: the ``model_template`` of every node/edge and its per-node/per-edge ``dynamics_params``;
- model templates: one JSON file per template in ``point_neuron_models_dir``, naming the NEST model and its parameters (as in :doc:`Brian2 <sonata_brian2>`);

``point_neuron_models_dir`` is the same property as in :ref:`components`, used for the templates of both neuron and edge (synapse) populations. It can be set per population, or once under ``components`` as a default that populations without their own value inherit.

Population types are ``nest_point`` for neurons, ``nest_chemical`` for synapses and ``virtual`` for inputs. The ``nest_`` names are specific to NEST because ``point_neuron`` and ``chemical`` are already defined for other simulators, with other fields, and a simulator that loads populations by type could pick up a NEST population by mistake. NEST SONATA defines no edge type, and calls the neuron types ``point_neuron`` and ``point_process``. Each node file also stores the type in ``/0/model_type``. Only spike-transmitting synapse models are supported: gap junctions, rate and diffusion connections and ``sic_connection`` are not.

Circuit Config
--------------

.. table::

    ============================== ============ ==================================================
    Property                       Requirement  Description
    ============================== ============ ==================================================
    type                           Mandatory    ``nest_point`` or ``virtual`` (nodes); ``nest_chemical`` (edges)
    point_neuron_models_dir        Conditional  Directory holding the population's model templates and NESTML sources.
                                                Mandatory for neuron and edge populations, either on the population or under ``components``.
    nest_version                   Mandatory    Version of NEST the circuit was built for, e.g. ``3.10.0``. Under ``components`` ``provenance``.
    nestml_version                 Optional     Version of NESTML used for the ``nestml:`` models, e.g. ``9.0.1``. Under ``components`` ``provenance``.
    ============================== ============ ==================================================

.. code-block:: json

    {
      "version": 2,
      "target_simulator": "NEST",
      "node_sets_file": "node_sets.json",
      "components": {"provenance": {"nest_version": "3.10.0"}},
      "networks": {
        "nodes": [
          {
            "nodes_file": "S1/nodes.h5",
            "populations": {
              "S1": {"type": "nest_point", "point_neuron_models_dir": "model_templates/S1"}
            }
          },
          {
            "nodes_file": "VPM/nodes.h5",
            "populations": {"VPM": {"type": "virtual"}}
          }
        ],
        "edges": [
          {
            "edges_file": "VPM__S1__chemical/edges.h5",
            "populations": {
              "VPM__S1__chemical": {
                "type": "nest_chemical",
                "point_neuron_models_dir": "model_templates/VPM__S1__chemical"
              }
            }
          }
        ]
      }
    }

``nest_version`` and ``nestml_version`` are in the ``provenance`` dictionary of ``components`` (see :ref:`components`). Parameter names and defaults can change between NEST and NESTML versions, so a circuit records the versions it was built for: ``nest_version`` is the NEST version, as in ``nest.__version__``, and ``nestml_version`` is the NESTML version used to compile the circuit's ``nestml:`` models, and is only set if there are any. A simulator that finds a different major or minor version in use, or no ``nest_version``, should warn and carry on. libsonata does not read or check them.

Model Templates
---------------

``<point_neuron_models_dir>/<model_template>.json``, shared by all nodes/edges with that ``model_template``:

.. table::

    ================== ============ =========================================================================
    Key                Requirement  Description
    ================== ============ =========================================================================
    model              Mandatory    ``nest:<model>`` (built-in NEST model) or ``nestml:<model>`` (compiled from ``<point_neuron_models_dir>/<model>.nestml`` on demand)
    params             Optional     Parameters set on every node/edge using the template. For synapses, ``weight`` and ``delay`` act as defaults for edges without values, and ``weight`` has the unit of ``syn_weight``, see :ref:`Edges <nest_edges>`.
    dynamics_params    Optional     Parameter name -> unit of the per-node/per-edge values read from ``/0/dynamics_params/<name>``. Units are informational; NEST uses the model's native units.
    ================== ============ =========================================================================

The names in ``params`` and ``dynamics_params`` are parameters of the NEST model, as NEST accepts them in ``Create`` and ``Connect``. ``nest.GetDefaults("<model>")`` lists them with their default values, together with some read-only entries such as ``recordables``, and the NEST model reference gives their units. For a ``nestml:`` model they are the names declared in the ``parameters`` and ``state`` blocks of the ``.nestml`` file, listed by ``nest.GetDefaults`` once the model is compiled and installed. NEST rejects any other name with an error when the network is built, and a reader should report the template and the model. Values are in the native units of the model; the units in ``dynamics_params`` are informational. libsonata does not check the names, since that needs NEST.

.. code-block:: json

    {
      "model": "nest:aeif_cond_alpha",
      "params": {"V_th": -50.4, "C_m": 200.0, "tau_w": 170.0},
      "dynamics_params": {"E_L": "mV", "g_L": "nS"}
    }

Nodes
-----

Neuron population, ``type`` ``nest_point``:

.. table::

    ================== =============================== ========== ===========================================================
    Group              Field                           Type       Description
    ================== =============================== ========== ===========================================================
    /0                 model_template                  utf8       Model template name
    /0                 model_type                      utf8       ``nest_point``
    /0/dynamics_params ...                             float32    Per-node values of the template's ``dynamics_params``
    /                  node_type_id                    int        Set to -1, or the source node type id
    ================== =============================== ========== ===========================================================

Values are applied in order: ``params``, then ``dynamics_params``.

``/0/model_type`` is mandatory, as for the node populations of :doc:`sonata_tech`, and equals the ``type`` of the population in the circuit config. NEST does not read it. It makes the node file self-describing and lets node sets select on it.

Virtual (input) population, ``type`` ``virtual``, with the same fields as the virtual population of :doc:`sonata_tech`:

.. table::

    ================== =============================== ========== ============= ===========================================================
    Group              Field                           Type       Requirement   Description
    ================== =============================== ========== ============= ===========================================================
    /0                 model_type                      utf8       Mandatory     ``virtual``
    /0                 model_template                  utf8       Optional      Normally omitted, see below
    /                  node_type_id                    int        Mandatory     Set to -1, or the source node type id
    ================== =============================== ========== ============= ===========================================================

When there is no applicable model template for a given model type (i.e. ``model_type=virtual``) the ``model_template`` field should be omitted. NEST creates no nodes for a virtual population and does not use a model template for it. Its spikes come from the ``synapse_replay`` inputs (see :ref:`Simulation Config <nest_inputs>`) and are delivered through the edge populations whose source it is.

.. _nest_edges:

Edges
-----

Synapse population, ``type`` ``nest_chemical``:

.. table::

    ================== =============================== ========== ===========================================================
    Group              Field                           Type       Description
    ================== =============================== ========== ===========================================================
    /0                 model_template                  utf8       Model template name
    /0                 syn_weight                      float32    Optional per-edge weight; its unit and sign depend on the postsynaptic model, see below
    /0                 delay                           float32    Optional per-edge delay (ms); clamped to the kernel resolution
    /0/dynamics_params ...                             float32    Per-edge values of the template's ``dynamics_params``, e.g. ``U``, ``u``, ``tau_rec``, ``tau_fac``, ``receptor_type``
    ================== =============================== ========== ===========================================================

The ``syn_spec`` of an edge is built from the template's ``model``, then ``params``, then ``syn_weight``/``delay``, then ``dynamics_params``.

NEST does not convert weights: ``syn_weight``, and ``weight`` in the ``params`` of a synapse template, are in the native unit of the postsynaptic model. It is pA for current-based models (``*_psc_*``, e.g. ``iaf_psc_alpha``), nS for conductance-based models (``*_cond_*``) and mV for models with delta synapses (``iaf_psc_delta``, ``izhikevich``), where the weight is the jump of the membrane potential. A weight cannot be interpreted without the postsynaptic model.

Excitatory weights are positive. Inhibitory weights are negative in current-based and delta models, and in conductance-based models with one synapse type (e.g. ``iaf_cond_alpha``), where a negative weight acts on the inhibitory conductance. The exception is conductance-based ``*_multisynapse`` models (e.g. ``aeif_cond_beta_multisynapse``): all weights are positive, NEST rejects negative ones, and the receptor port selected with ``receptor_type`` carries the sign through its reversal potential ``E_rev``.

.. _nest_inputs:

Simulation Config
-----------------

``run`` requires ``dt``, ``tstop`` and ``random_seed``. Spikes are given as :ref:`synapse_replay <spikes-synapse-replay>` inputs, with the meaning of :doc:`sonata_simulation`: the ``node_set`` of an input is the set of nodes affected by it, i.e. the targets. The ``spike_file`` is a SONATA spike file: ``/spikes/<population>/node_ids`` and ``/spikes/<population>/timestamps`` for each population it covers. The spikes of each population are delivered through every edge population whose source is that population, to the targets in ``node_set``, with the synapse model, weight and delay of each edge. The population can be a ``virtual`` population, or a ``nest_point`` population whose spikes are then replayed through its own edges on top of its simulated activity. Spike times outside ``[delay, delay + duration)`` are dropped, and the others are not shifted. An input whose ``node_set`` contains no target of those edge populations delivers nothing, which NEST reports as an error. Other input modules are not supported.

.. code-block:: json

    {
      "network": "circuit_config.json",
      "target_simulator": "NEST",
      "run": {"dt": 0.1, "tstop": 1000.0, "random_seed": 0},
      "inputs": {
        "vpm_spikes": {
          "input_type": "spikes",
          "module": "synapse_replay",
          "spike_file": "inputs/vpm_spikes.h5",
          "node_set": "S1",
          "delay": 0.0,
          "duration": 1000.0
        }
      }
    }

Conversion from NEST SONATA
---------------------------

Circuits in the `NEST SONATA format <https://nest-simulator.readthedocs.io/en/stable/nest_sonata/nest_sonata_guide.html>`_ map as follows:

.. list-table::
    :header-rows: 1
    :widths: 45 55

    * - NEST SONATA
      - SONATA extension
    * - ``components`` ``point_neuron_models_dir``
      - ``point_neuron_models_dir`` of each neuron population; its node types' templates and ``.nestml`` sources are written there
    * - ``components`` ``synaptic_models_dir``
      - ``point_neuron_models_dir`` of each edge population; its edge types' templates and ``.nestml`` sources are written there
    * - node types CSV row (``node_type_id``, ``model_template``, ``dynamics_params`` JSON)
      - one model template per node type, with the JSON as ``params``; ``/0/model_template`` set from ``node_type_id``
    * - ``model_type`` ``point_neuron`` or ``point_process``
      - population ``type`` ``nest_point``, and ``/0/model_type`` ``nest_point``
    * - ``model_type`` ``virtual``
      - population ``type`` ``virtual`` and ``/0/model_type`` ``virtual``
    * - edges (no ``model_type``)
      - population ``type`` ``nest_chemical``
    * - edge types CSV row (``edge_type_id``, ``model_template``, ``dynamics_params`` JSON, ``delay``, ``syn_weight``)
      - one model template per edge type (``nest:`` prefix added); ``/0/model_template`` set from ``edge_type_id``; ``params`` are the CSV ``delay``/``syn_weight`` (as ``delay``/``weight``) and the other CSV columns that NEST sets as parameters of the synapse model, with the parameters of the ``dynamics_params`` JSON on top, which win on conflicts as in NEST
    * - other node types CSV columns (e.g. ``ei``, ``model_name``)
      - per-node attributes under ``/0``: a column that is a number in every type is stored as a number, any other as an ``@library`` enumeration; ``NULL`` and empty values are dropped, and types without a value get NaN or an empty string; a dataset already stored per node in the HDF5 file takes precedence; NEST does not use them
    * - other edge types CSV columns (e.g. ``source_query``, ``target_query``)
      - per-edge attributes under ``/0``, with the same rules; NEST does not use them. A column that is a parameter of the synapse model is not an attribute but a template ``params`` entry, see the row above
    * - per-edge ``delay``, ``syn_weight``
      - unchanged
    * - other per-edge datasets overriding the synapse JSON
      - moved to ``/0/dynamics_params``
    * - ``inputs`` (``input_file``, ``node_set``)
      - ``synapse_replay`` input (``spike_file``, ``node_set``). NEST SONATA names the source population in ``node_set``, whereas here it selects the targets: the converter adds a node set ``<population>_targets`` to ``node_sets.json``, the union of the target populations of the edge populations that leave the source population, and uses it. A spike file in the legacy layout (``/spikes/gids`` and ``/spikes/timestamps``) is rewritten to ``/spikes/<population>/node_ids`` and ``timestamps``, where the population is the one the NEST SONATA ``node_set`` names
    * - ``run`` (``dt``, ``tstop``)
      - simulation config ``run`` (plus ``random_seed``)
    * - node sets files of the circuit and the simulation config
      - merged into one ``node_sets.json`` that also holds one set per population, and the ``<population>_targets`` set of each input
    * - ``conditions``, ``output``, ``reports``, ``run`` ``nsteps_block``
      - not converted: NEST ignores them, and the converter logs a warning that lists the ones it drops

Future Work
-----------

* NESTML synapses co-generated with a postsynaptic neuron (e.g. STDP) are not supported.
* Current inputs and recordings are not yet defined for NEST.

Example: Full Conversion
------------------------

This example converts a small NEST SONATA circuit and shows every input and output file. The circuit has a population ``exc`` of four neurons with two node types (a built-in NEST model and a NESTML model), a virtual population ``input`` of two spike sources, and two edge populations: ``input_to_exc`` (``static_synapse``) and ``exc_to_exc`` (``tsodyks2_synapse`` with per-edge values).

NEST SONATA input
~~~~~~~~~~~~~~~~~

.. code-block:: text

    nest_sonata/
    ├── circuit_config.json
    ├── components/
    │   ├── point_neuron_models/
    │   │   ├── adaptive.nestml
    │   │   └── exc_iaf.json
    │   └── synaptic_models/
    │       └── stp.json
    ├── inputs/spikes.h5
    └── network/
        ├── nodes.h5                 node_types.csv
        ├── input_nodes.h5           input_node_types.csv
        ├── input_edges.h5           input_edge_types.csv
        └── exc_edges.h5             exc_edge_types.csv

``circuit_config.json`` holds the model directories (``components``), the run parameters and the spike input, besides the network files:

.. code-block:: json

    {
      "manifest": {"$BASE_DIR": "."},
      "target_simulator": "NEST",
      "run": {"dt": 0.1, "tstop": 100.0},
      "components": {
        "point_neuron_models_dir": "$BASE_DIR/components/point_neuron_models",
        "synaptic_models_dir": "$BASE_DIR/components/synaptic_models"
      },
      "inputs": {
        "input_spikes": {
          "input_type": "spikes",
          "module": "h5",
          "input_file": "$BASE_DIR/inputs/spikes.h5",
          "node_set": "input"
        }
      },
      "networks": {
        "nodes": [
          {"nodes_file": "$BASE_DIR/network/nodes.h5", "node_types_file": "$BASE_DIR/network/node_types.csv"},
          {"nodes_file": "$BASE_DIR/network/input_nodes.h5", "node_types_file": "$BASE_DIR/network/input_node_types.csv"}
        ],
        "edges": [
          {"edges_file": "$BASE_DIR/network/input_edges.h5", "edge_types_file": "$BASE_DIR/network/input_edge_types.csv"},
          {"edges_file": "$BASE_DIR/network/exc_edges.h5", "edge_types_file": "$BASE_DIR/network/exc_edge_types.csv"}
        ]
      }
    }

The type CSVs give one model per type. ``dynamics_params`` names a JSON file in the model directory, ``NULL`` means none:

.. code-block:: text

    node_types.csv
    node_type_id model_type    model_template     dynamics_params
    0            point_process nest:iaf_psc_alpha exc_iaf.json
    1            point_neuron  nestml:adaptive    NULL

    input_node_types.csv
    node_type_id model_type model_template dynamics_params
    0            virtual    NULL           NULL

    input_edge_types.csv
    edge_type_id model_template  dynamics_params delay syn_weight
    100          static_synapse  NULL            1.0   50.0

    exc_edge_types.csv
    edge_type_id model_template   dynamics_params delay syn_weight
    101          tsodyks2_synapse stp.json        1.5   2.0

.. code-block:: text

    components/point_neuron_models/exc_iaf.json
    {"V_th": -50.0, "C_m": 200.0}

    components/synaptic_models/stp.json
    {"U": 0.5, "u": 0.5, "tau_rec": 500.0, "tau_fac": 20.0}

The HDF5 files carry the type ids and the per-node and per-edge values (``nodes.h5``, ``input_edges.h5`` and ``exc_edges.h5``; ``input_nodes.h5`` only has ``node_id`` and ``node_type_id``):

.. code-block:: text

    nodes.h5
    /nodes/exc/node_type_id                 [0, 0, 1, 1]
    /nodes/exc/0/dynamics_params/V_m        [-70.0, -68.0, -65.0, -72.0]

    input_edges.h5
    /edges/input_to_exc/source_node_id      [0, 1, 1]
    /edges/input_to_exc/target_node_id      [0, 2, 3]
    /edges/input_to_exc/edge_type_id        [100, 100, 100]

    exc_edges.h5
    /edges/exc_to_exc/source_node_id        [0, 1, 2]
    /edges/exc_to_exc/target_node_id        [1, 2, 3]
    /edges/exc_to_exc/edge_type_id          [101, 101, 101]
    /edges/exc_to_exc/0/syn_weight          [2.0, 3.0, 1.5]
    /edges/exc_to_exc/0/delay               [1.5, 2.0, 1.0]
    /edges/exc_to_exc/0/U                   [0.4, 0.5, 0.6]
    /edges/exc_to_exc/0/receptor_type       [1, 1, 2]

SONATA-extension output
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: text

    nest_sonata_extension/
    ├── circuit_config.json
    ├── simulation_config.json
    ├── node_sets.json
    ├── inputs/spikes.h5
    ├── model_templates/
    │   ├── exc/
    │   │   ├── iaf_psc_alpha_type0.json
    │   │   ├── adaptive_type1.json
    │   │   └── adaptive.nestml
    │   ├── input_to_exc/static_synapse.json
    │   └── exc_to_exc/tsodyks2_synapse.json
    └── network/
        ├── nodes.h5
        ├── input_nodes.h5
        ├── input_edges.h5
        └── exc_edges.h5

The circuit config only describes the network. Each population has its ``type`` and its own ``point_neuron_models_dir``; the type CSVs and the model directories under ``components`` are gone, and ``components`` keeps the NEST version:

.. code-block:: json

    {
      "version": 2,
      "target_simulator": "NEST",
      "manifest": {"$BASE_DIR": "."},
      "node_sets_file": "$BASE_DIR/node_sets.json",
      "components": {"provenance": {"nest_version": "3.10.0"}},
      "networks": {
        "nodes": [
          {
            "nodes_file": "$BASE_DIR/network/nodes.h5",
            "populations": {
              "exc": {"type": "nest_point", "point_neuron_models_dir": "$BASE_DIR/model_templates/exc"}
            }
          },
          {
            "nodes_file": "$BASE_DIR/network/input_nodes.h5",
            "populations": {"input": {"type": "virtual"}}
          }
        ],
        "edges": [
          {
            "edges_file": "$BASE_DIR/network/input_edges.h5",
            "populations": {
              "input_to_exc": {"type": "nest_chemical", "point_neuron_models_dir": "$BASE_DIR/model_templates/input_to_exc"}
            }
          },
          {
            "edges_file": "$BASE_DIR/network/exc_edges.h5",
            "populations": {
              "exc_to_exc": {"type": "nest_chemical", "point_neuron_models_dir": "$BASE_DIR/model_templates/exc_to_exc"}
            }
          }
        ]
      }
    }

The run parameters and the spike input move to the simulation config. The input becomes a ``synapse_replay`` input whose ``spike_file`` is the copied file, active for the whole run. Its ``node_set`` is ``input_targets``, the targets of the edges that leave ``input``, which are in ``exc``:

.. code-block:: json

    {
      "version": 2,
      "manifest": {"$BASE_DIR": "."},
      "network": "$BASE_DIR/circuit_config.json",
      "target_simulator": "NEST",
      "run": {"dt": 0.1, "tstop": 100.0, "random_seed": 0},
      "inputs": {
        "input_spikes": {
          "input_type": "spikes",
          "module": "synapse_replay",
          "spike_file": "$BASE_DIR/inputs/spikes.h5",
          "node_set": "input_targets",
          "delay": 0.0,
          "duration": 100.0
        }
      }
    }

Each type row becomes one model template. The CSV model gets the ``nest:`` prefix if it has none, and the type JSON becomes ``params``. For a synapse, the CSV ``delay`` and ``syn_weight`` become the ``delay`` and ``weight`` defaults, and so does every other CSV column that NEST sets as a parameter of the synapse model; the JSON parameters overwrite them where both give the same name. ``dynamics_params`` lists the per-node and per-edge values stored in the HDF5 files, with their units:

.. code-block:: text

    model_templates/exc/iaf_psc_alpha_type0.json
    {
      "model": "nest:iaf_psc_alpha",
      "params": {"V_th": -50.0, "C_m": 200.0},
      "dynamics_params": {"V_m": "mV"}
    }

    model_templates/exc/adaptive_type1.json
    {
      "model": "nestml:adaptive",
      "dynamics_params": {"V_m": "mV"}
    }

    model_templates/input_to_exc/static_synapse.json
    {
      "model": "nest:static_synapse",
      "params": {"delay": 1.0, "weight": 50.0}
    }

    model_templates/exc_to_exc/tsodyks2_synapse.json
    {
      "model": "nest:tsodyks2_synapse",
      "params": {"U": 0.5, "u": 0.5, "tau_rec": 500.0, "tau_fac": 20.0, "delay": 1.5, "weight": 2.0},
      "dynamics_params": {"U": "1", "receptor_type": "1", "u": "1"}
    }

``node_sets.json`` has one node set per population, and a compound node set ``input_targets`` with the target populations of the edges that leave ``input``: ``{"exc": {"population": "exc"}, "input": {"population": "input"}, "input_targets": ["exc"]}``.

In the HDF5 files, ``node_type_id`` and ``edge_type_id`` are kept and ``/0/model_template`` (and ``/0/model_type`` for nodes) are added as ``@library`` enumerations. The per-edge values that override the synapse model move into ``/0/dynamics_params``:

.. code-block:: text

    nodes.h5
    /nodes/exc/node_type_id                       [0, 0, 1, 1]
    /nodes/exc/0/model_type                       [0, 0, 0, 0]
    /nodes/exc/0/@library/model_type              ["nest_point"]
    /nodes/exc/0/model_template                   [1, 1, 0, 0]
    /nodes/exc/0/@library/model_template          ["adaptive_type1", "iaf_psc_alpha_type0"]
    /nodes/exc/0/dynamics_params/V_m              [-70.0, -68.0, -65.0, -72.0]

    input_nodes.h5
    /nodes/input/node_type_id                     [0, 0]
    /nodes/input/0/model_type                     [0, 0]
    /nodes/input/0/@library/model_type            ["virtual"]

    input_edges.h5
    /edges/input_to_exc/edge_type_id              [100, 100, 100]
    /edges/input_to_exc/0/model_template          [0, 0, 0]
    /edges/input_to_exc/0/@library/model_template ["static_synapse"]

    exc_edges.h5
    /edges/exc_to_exc/edge_type_id                [101, 101, 101]
    /edges/exc_to_exc/0/model_template            [0, 0, 0]
    /edges/exc_to_exc/0/@library/model_template   ["tsodyks2_synapse"]
    /edges/exc_to_exc/0/syn_weight                [2.0, 3.0, 1.5]
    /edges/exc_to_exc/0/delay                     [1.5, 2.0, 1.0]
    /edges/exc_to_exc/0/dynamics_params/U         [0.4, 0.5, 0.6]
    /edges/exc_to_exc/0/dynamics_params/u         [0.4, 0.5, 0.6]
    /edges/exc_to_exc/0/dynamics_params/receptor_type [1, 1, 2]

Summary of the changes
~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
    :header-rows: 1
    :widths: 40 60

    * - NEST SONATA
      - SONATA extension (in this example)
    * - ``components`` ``point_neuron_models_dir`` and ``synaptic_models_dir``
      - one ``point_neuron_models_dir`` per population: ``model_templates/exc``, ``model_templates/input_to_exc``, ``model_templates/exc_to_exc``
    * - no NEST version
      - ``components`` ``provenance`` ``nest_version``
    * - ``node_types_file`` and ``edge_types_file`` CSVs
      - removed; each row is one template JSON, and ``node_type_id``/``edge_type_id`` are kept in the HDF5 files
    * - ``model_type`` ``point_process`` (node type 0)
      - population ``type`` ``nest_point``, and ``/0/model_type`` ``nest_point``; ``virtual`` stays ``virtual`` in both
    * - edge populations without a type
      - ``type`` ``nest_chemical``
    * - ``model_template`` in the CSV (``tsodyks2_synapse``)
      - ``model`` with the ``nest:`` prefix added (``nest:tsodyks2_synapse``); ``nestml:`` models are kept as they are
    * - several types in one population
      - template names get a ``_type<id>`` suffix (``iaf_psc_alpha_type0``, ``adaptive_type1``); a population with one type uses the model name (``static_synapse``)
    * - ``dynamics_params`` JSON of a type
      - ``params`` of the template
    * - ``adaptive.nestml`` in the node model directory
      - copied next to the templates of every population that uses it (``model_templates/exc``)
    * - CSV ``delay``, ``syn_weight`` and other CSV columns that NEST sets as synapse parameters
      - ``params`` ``delay``, ``weight`` and the parameters of the template (defaults for edges without values); the ``dynamics_params`` JSON overwrites them on conflicts
    * - per-edge ``syn_weight`` and ``delay``
      - unchanged in ``/0``
    * - per-node ``/0/dynamics_params/V_m``
      - unchanged, and declared with its unit in the template's ``dynamics_params``
    * - per-edge ``U`` and ``receptor_type`` in ``/0``
      - moved to ``/0/dynamics_params``; ``u`` is added with the values of ``U``, because the template declares ``u`` and NEST SONATA exporters set the initial ``u`` to the per-edge ``U``
    * - ``inputs`` ``input_spikes`` (``module`` ``h5``, ``input_file``)
      - simulation config ``synapse_replay`` input (``spike_file``), with ``node_set`` ``input_targets``, ``delay`` 0 and ``duration`` equal to ``tstop``
    * - ``run`` ``dt``, ``tstop``
      - simulation config ``run``, plus ``random_seed``
    * - node sets
      - ``node_sets.json`` with one set per population, plus ``input_targets``
