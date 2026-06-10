MOD Files and Mechanisms
========================

NEURON simulations rely on **NMODL** (NEURON Model Description Language) files,
commonly referred to as **MOD files**, to define custom membrane mechanisms such
as ion channels, synapses, and point-process models.
In the SONATA ecosystem these files are compiled into shared libraries
(usually via ``nrnivmodl``) before the simulation starts.

Mechanism Types
---------------

The following categories of mechanisms are typically implemented as MOD files:

- **Ion channels** — voltage-gated or ligand-gated channels (e.g. ``Na``,
  ``K``, ``Ca``) inserted into sections via the HOC template or internal
  model-loading logic.
- **Synapses** — chemical synapse models (e.g. ``ExpSyn``, ``ProbAMPANMDA``,
  ``GluSynapse``) instantiated as point processes on dendritic or somatic
  segments.
- **Gap junctions** — electrical synapse models that use ``POINTER``
  variables to couple the membrane potential of two cells.
- **Current / clamp sources** — stimulus mechanisms such as ``IClamp``,
  ``SEClamp``, or custom current sources.
- **Reports / utility mechanisms** — auxiliary mechanisms for recording
  variables or performing arithmetic (e.g. ``ALU``, ``SonataReport``).

.. note::
  
   The SONATA circuit folder only contains the MOD files for ion channels, synapses, and gap junctions. The Current / clamp sources and Reports / utility mechanisms are not included in the circuit folder and added by simulator during simulation setup.
   The standard NEURON built-in mechanisms mentioned below such as ``ExpSyn``, ``Exp2Syn``, ``IntFire1``, ``IntFire2``, ``IntFire4``, ``IClamp`` and ``SEClamp`` are available by default via NEURON simulator (used by both BlueCelluLab and Neurodamus), and do not require MOD files in the circuits MOD directory.

Standard NEURON Mechanisms
--------------------------

In addition to custom MOD files, standard NEURON built-in mechanisms are
valid and may be referenced directly by name:

- ``ExpSyn`` — exponential synapse
- ``Exp2Syn`` — double-exponential synapse
- ``IntFire1``, ``IntFire2``, ``IntFire4`` — integrate-and-fire point neurons
- ``IClamp`` — current clamp
- ``SEClamp`` — single-electrode voltage clamp

Custom mechanisms must follow the same naming conventions as their MOD file
``SUFFIX`` or ``POINT_PROCESS`` declarations.

Specifying MOD Files in the Circuit Config
------------------------------------------

The path to the directory containing MOD files is declared in the
``circuit_config.json`` or ``simulation_config.json`` via the
``mechanisms_dir`` field:

.. code-block:: json

   {
       "components": {
           "mechanisms_dir": "/path/to/mod/files"
       }
   }

Simulation implementations (e.g. Neurodamus, BlueCelluLab) are responsible
for compiling these files (typically with ``nrnivmodl``) and loading the
resulting shared library before any cell or synapse instantiation occurs.

Synapse Model Override
----------------------

By default excitatory and inhibitory synapses are instantiated with helper
HOC files derived from the base synapse model names
(``AMPANMDAHelper.hoc`` / ``GABAABHelper.hoc``).
The ``connection_override`` mechanism in the simulation configuration allows
switching to a different synapse model via the ``modoverride`` field:

.. code-block:: json

   "connection_override" : [
      {
          "name": "my_override",
          "modoverride": "GluSynapse"
      }
   ]

When ``modoverride`` is set to ``"GluSynapse"``, the simulator loads
``GluSynapseHelper.hoc`` instead of the default helper. The helper file is
responsible for creating the synapse point process and initialising any
extra parameters that the custom MOD file requires (e.g. plasticity-related
variables). All parameters expected by the new model must be present in the
SONATA edges file; otherwise instantiation will fail.

Conventions and Best Practices
--------------------------------

- **Reproducible random numbers**: MOD files that use random-number generators
  (e.g. for stochastic synapses or channel noise) should expose the RNG stream
  ID as an assignable parameter so that the simulator can initialise it in a
  deterministic, per-synapse manner.
- **Global variables**: Global variables declared in MOD files (``GLOBAL``
  keyword) affect every instance of that mechanism. They can be overridden
  via HOC snippets (e.g. in ``synapse_configure``), but care must be taken
  because the change applies universally.
- **POINTER mechanisms**: Gap-junction and continuous-connection models that
  use ``POINTER`` variables require the simulator to explicitly bind the
  pointer to the target variable (usually another cell's membrane potential)
  after all cells have been instantiated.
- **Compilation order**: MOD files may depend on each other. The compilation
  tool (``nrnivmodl``) resolves most dependencies automatically, but
  ``POINTER``-based models should be checked for correct symbol resolution.

Relationship to HOC Templates
------------------------------

MOD files define the low-level biophysical mechanisms, while the
:doc:`HOC template <hoc-emodel>` defines how those mechanisms are inserted
into the cell morphology and how their parameters are mapped from the SONATA
node and edge files. For example:

- The HOC template may insert ``hh`` or custom channels into specific
  ``SectionList``\ s (``somatic``, ``axonal``, ``apic``, etc.).
- Synapse helpers (``<Suffix>Helper.hoc``) bridge the edge-file parameters
  and the MOD file ``PARAMETER``\ s, ensuring that each synapse instance
  receives the correct conductance, reversal potential, and timing constants.

References
----------

- `NMODL documentation <https://nrn.readthedocs.io/en/latest/nmodl/index.html>`_
- `NEURON HOC reference <https://nrn.readthedocs.io/en/latest/hoc/index.html>`_
- `SONATA extension documentation <https://sonata-extension.readthedocs.io/en/latest/sonata_overview.html>`_
- `SONATA specification (Allen Institute) <https://github.com/AllenInstitute/sonata/blob/main/docs/SONATA_DEVELOPER_GUIDE.md>`_
