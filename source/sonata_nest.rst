.. _sonata_nest:

Describing NEST Models with SONATA
==================================

Overview
--------

`NEST <https://nest-simulator.readthedocs.io/>`_ is a simulator for networks of point neurons.
This document describes how such a network is encoded in SONATA so it can be read by `libsonata` and instantiated in NEST through the PyNEST ``Create``/``Connect`` API.

A NEST circuit uses ``target_simulator: "NEST"`` and is described by:

- the circuit config: each population's ``type`` and ``point_neuron_models_dir``;
- the nodes/edges files: the ``model_template`` of every node/edge and its per-node/per-edge ``dynamics_params``;
- model templates: one JSON file per template in ``point_neuron_models_dir``, naming the NEST model and its parameters (as in :doc:`Brian2 <sonata_brian2>`);
- the simulation config: ``run`` and the spike inputs driving virtual populations.

``point_neuron_models_dir`` is the same property as in :ref:`components`, used for the templates of both neuron and edge (synapse) populations.
It can be set per population, or once under ``components`` as a default that populations without their own value inherit.

Population types are the NEST SONATA ``model_type`` values: ``point_neuron`` and ``point_process`` (interchangeable) for neurons, ``virtual`` for inputs.
NEST SONATA defines no edge type, so edges use the default ``chemical``.
With ``target_simulator: "NEST"`` these populations follow this page, not the field tables of :doc:`sonata_tech`.

Circuit Config
--------------

.. table::

    ============================== ============ ==================================================
    Property                       Requirement  Description
    ============================== ============ ==================================================
    type                           Mandatory    ``point_neuron``, ``point_process`` or ``virtual`` (nodes); ``chemical`` (edges)
    point_neuron_models_dir        Conditional  Directory holding the population's model templates and NESTML sources.
                                                Mandatory for neuron and edge populations, either on the population or under ``components``.
    ============================== ============ ==================================================

Template names (``<model_template>.json``, ``<model>.nestml``) must be unique within a directory.
Populations that share a directory, including through the ``components`` default, must not reuse a name for different templates.

.. code-block:: json

    {
      "version": 2,
      "target_simulator": "NEST",
      "node_sets_file": "node_sets.json",
      "networks": {
        "nodes": [
          {
            "nodes_file": "S1/nodes.h5",
            "populations": {
              "S1": {"type": "point_neuron", "point_neuron_models_dir": "model_templates/S1"}
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
                "type": "chemical",
                "point_neuron_models_dir": "model_templates/VPM__S1__chemical"
              }
            }
          }
        ]
      }
    }

Model Templates
---------------

``<point_neuron_models_dir>/<model_template>.json``, shared by all nodes/edges with that ``model_template``:

.. table::

    ================== ============ =========================================================================
    Key                Requirement  Description
    ================== ============ =========================================================================
    model              Mandatory    ``nest:<model>`` (built-in NEST model) or ``nestml:<model>`` (compiled from ``<point_neuron_models_dir>/<model>.nestml`` on demand)
    params             Optional     Parameters set on every node/edge using the template. For synapses, ``weight`` and ``delay`` act as defaults for edges without values.
    dynamics_params    Optional     Parameter name -> unit of the per-node/per-edge values read from ``/0/dynamics_params/<name>``. Units are informational; NEST uses the model's native units.
    ================== ============ =========================================================================

.. code-block:: json

    {
      "model": "nest:aeif_cond_alpha",
      "params": {"V_th": -50.4, "C_m": 200.0, "tau_w": 170.0},
      "dynamics_params": {"E_L": "mV", "g_L": "nS"}
    }

Nodes
-----

Neuron population, ``type`` ``point_neuron`` or ``point_process``:

.. table::

    ================== =============================== ========== ===========================================================
    Group              Field                           Type       Description
    ================== =============================== ========== ===========================================================
    /0                 model_template                  utf8       Model template name (an ``@library`` enumeration is recommended)
    /0/dynamics_params ...                             float32    Per-node values of the template's ``dynamics_params``
    /                  node_type_id                    int        Set to -1, or the source node type id
    ================== =============================== ========== ===========================================================

Values are applied in order: ``params``, then ``dynamics_params``.

Virtual (input) population, ``type`` ``virtual``: only ``node_type_id`` and an empty ``/0`` group are required.
Its nodes are created as ``spike_train_injector`` and emit the spikes of the ``synapse_replay`` inputs targeting them.

Edges
-----

Synapse population, ``type`` ``chemical``:

.. table::

    ================== =============================== ========== ===========================================================
    Group              Field                           Type       Description
    ================== =============================== ========== ===========================================================
    /0                 model_template                  utf8       Model template name (an ``@library`` enumeration is recommended)
    /0                 syn_weight                      float32    Optional per-edge weight
    /0                 delay                           float32    Optional per-edge delay (ms); clamped to the kernel resolution
    /0/dynamics_params ...                             float32    Per-edge values of the template's ``dynamics_params``, e.g. ``U``, ``u``, ``tau_rec``, ``tau_fac``, ``receptor_type``
    ================== =============================== ========== ===========================================================

The ``syn_spec`` of an edge is built from the template's ``model``, then ``params``, then ``syn_weight``/``delay``, then ``dynamics_params``.

.. _nest_inputs:

Simulation Config
-----------------

``run`` requires ``dt``, ``tstop`` and ``random_seed``.
Spikes of virtual populations are given as :ref:`synapse_replay <spikes-synapse-replay>` inputs whose ``node_set`` selects ``virtual`` nodes; spike times outside ``[delay, delay + duration)`` are dropped.
Other input modules are not supported.

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
          "node_set": "VPM",
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
    * - ``model_type`` ``point_neuron``, ``point_process``, ``virtual``
      - population ``type``, unchanged
    * - edges (no ``model_type``)
      - population ``type`` ``chemical``
    * - edge types CSV row (``edge_type_id``, ``model_template``, ``dynamics_params`` JSON, ``delay``, ``syn_weight``)
      - one model template per edge type (``nest:`` prefix added), with the JSON as ``params`` and the CSV ``delay``/``syn_weight`` as ``params`` ``delay``/``weight``; ``/0/model_template`` set from ``edge_type_id``
    * - per-edge ``delay``, ``syn_weight``
      - unchanged
    * - other per-edge datasets overriding the synapse JSON
      - moved to ``/0/dynamics_params``
    * - ``inputs`` (``input_file``, ``node_set``)
      - ``synapse_replay`` input (``spike_file``, ``node_set``)
    * - ``run`` (``dt``, ``tstop``)
      - simulation config ``run`` (plus ``random_seed``)

Future Work
-----------

* NESTML synapses co-generated with a postsynaptic neuron (e.g. STDP) are not supported.
* Current inputs and recordings are not yet defined for NEST.
