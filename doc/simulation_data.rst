.. _doc_simulation_data:

Creation of :class:`.SimulationData` objects
============================================

The data of simulations to be validated need to be represented by objects
of the  :class:`.SimulationData` type.
The  :class:`.SimulationData` objects are consisting of information about the
simulation and the system. This information is collected in objects of different
classes, namely

* :obj:`.SimulationData.units` of type :class:`.UnitData`:
  Information on the units used by the simulation program.
* :obj:`.SimulationData.ensemble` of type :class:`.EnsembleData`:
  Information describing the sampled ensemble.
* :obj:`.SimulationData.system` of type :class:`.SystemData`:
  Information on the system (numbers of atoms, molecules, constraints, etc.).
* :obj:`.SimulationData.observables` of type :class:`.ObservableData`:
  Trajectories of observables along the simulation.
* :obj:`.SimulationData.trajectory` of type :class:`.TrajectoryData`:
  Position / velocity / force trajectories along the simulation.
* :obj:`.SimulationData.dt` of type :code:`float`:
  The time step at which the simulation was performed.

The :class:`.SimulationData` objects can either be constructed
directly from arrays and numbers, or (partially) automatically via parsers.

Create SimulationData objects from python data
----------------------------------------------

Example usage, system of 900 water molecules in GROMACS units simulated in NVT:
::

   import numpy as np
   import physical_validation

   simulation_data = physical_validation.data.SimulationData()

   num_molecules = 900
   simulation_data.system = physical_validation.data.SystemData(
       # Each water molecule has three atoms
       natoms=num_molecules * 3,
       # Each molecule has three constraints
       nconstraints=num_molecules * 3,
       # In this simulation, translational center of mass motion was removed
       ndof_reduction_tra=3,
       # Rotational center of mass motion was not removed
       ndof_reduction_rot=0,
       # Repeat weight of one oxygen and two hydrogen atoms 900 times
       mass=np.tile([15.9994, 1.008, 1.008], num_molecules),
       # Denotes the first atom of each molecules: [0, 3, 6, ...]
       molecule_idx=np.linspace(0, num_molecules * 3, num_molecules, endpoint=False, dtype=int),
       # Each molecule has three constraints
       nconstraints_per_molecule=3 * np.ones(num_molecules),
   )

   # Set GROMACS units
   simulation_data.units = physical_validation.data.UnitData.units("GROMACS")

   # Simulation was performed under NVT conditions
   simulation_data.ensemble = physical_validation.data.EnsembleData(
       ensemble='NVT',
       natoms=num_molecules * 3,
       volume=3.01125 ** 3,
       temperature=298.15,
   )

   # This snippet is assuming that `kin_ene`, `pot_ene` and `tot_ene` are lists
   # or numpy arrays filled with the time series of kinetic, potential and total energy
   # of a simulation run. These might be obtained, e.g., from the python
   # API of a simulation code, or from other python-based analysis tools.
   simulation_data.observables = physical_validation.data.ObservableData(
       kinetic_energy=kin_ene,
       potential_energy=pot_ene,
       total_energy=tot_ene,
   )

   # We are further assuming that `positions` and `velocities` are arrays
   # of shape (number of frames) x (number of atoms) x 3, where the last
   # number stands for the 3 spatial dimensions. Again, these arrays would
   # most likely have been obtained from a python interface of the simulation
   # package or from other python-based analysis tools
   simulation_data.trajectory = physical_validation.data.TrajectoryData(
       position=positions,
       velocity=velocities,
   )

Package-specific instructions
-----------------------------

GROMACS
~~~~~~~
GROMACS does not offer a well-established Python interface to read out
energies or trajectories. :code:`physical_validation` therefore offers a parser,
which will return a fully populated :class:`.SimulationData` object by
reading in GROMACS input and output files.

The :class:`.GromacsParser` takes the GROMACS input files :code:`mdp` (run options)
and :code:`top` (topology file) to read the details about the system, the ensemble
and the time step. The observable trajectory is extracted from an :code:`edr`
(binary energy trajectory), while the position and velocity trajectory can
be read either from a :code:`trr` (binary trajectory) or a :code:`gro` (ASCII trajectory)
file. The constructor optionally takes the path to a gromacs binary as well
as the path to the topology library as inputs. The first is necessary to
extract information from binary files (using :code:`gmx energy` and :code:`gmx dump`),
while the second becomes necessary if the :code:`top` file contains :code:`#include` statements
which usually rely on GROMACS environment variables. The parser is able to
find GROMACS installations which are in the path (e.g. after sourcing the
:code:`GMXRC` file) and the corresponding topology library automatically.

Example usage:
::

   import physical_validation

   parser = physical_validation.data.GromacsParser()

   res = parser.get_simulation_data(
        mdp='mdout.mdp',
        top='system.top',
        gro='system.gro',
        edr='system.edr'
   )

.. note:: Always double-check the results received from the automatic parser.
   Since this is not an official GROMACS tool, it is very likely that some
   special cases or changes in recent versions might not be interpreted
   correctly.

LAMMPS
~~~~~~
To analyze simulations performed with LAMMPS, we strongly suggest using its
Python interface `Pizza.py <https://pizza.sandia.gov/index.html>`_ to create
a SimulationData object as explained in `Create SimulationData objects from python data`_.
Note that :class:`.UnitData` offers access to a UnitData
object representing the LAMMPS :code:`real` units by using
:code:`.data.UnitData.units("LAMMPS real")`.

As an alternative, :code:`physical_validation` ships with a LAMMPS parser, which tries
to read part of the system information, the observable and position / velocity
trajectories from LAMMPS output files.

Example usage:
::

   import physical_validation

   parser = physical_validation.data.LammpsParser()

   res = parser.get_simulation_data(
       # The LAMMPS parser cannot infer the ensemble from the LAMMPS files,
       # so we pass an EnsembleData object with the information matching the simulation
       ensemble=physical_validation.data.EnsembleData(
           ensemble="NVT",
           natoms=900,
           volume=20**3,
           temperature=300
       ),
       in_file=dir_1 + '/water.in',
       log_file=dir_1 + '/log.lammps',
       data_file=dir_1 + '/water.lmp',
       dump_file=dir_1 + '/dump.atom'
   )

.. warning:: The LAMMPS parser is in an early development stage. It 
   is part of the :code:`physical_validation`
   package in the hope that it is helpful to someone, but it is very
   likely to go wrong in a number of cases. Please check any object data
   create by the LAMMPS parser carefully.

Flatfile parser
---------------

For MD packages not supported by the package-specific parsers, it is possible
to create the :class:`.SimulationData` objects via the
:class:`.FlatfileParser`. This parser fills the
:obj:`.SimulationData.trajectory` object via 3-dimensional ASCII files
containing the position and velocity trajectories, and the
:obj:`.SimulationData.observables` via 1-dimensional ASCII files containing
the trajectories for the observables of interest. As the details on the
units, the simulated system and the sampled ensemble can not easily be read
from such files, this information has to be provided by the user by passing
objects of the respective data structures. See
:attr:`.FlatfileParser.get_simulation_data` for more details on the
:class:`.SimulationData` creation via the flat file parser, and
:ref:`simulationdata_details` for details on which test requires which
information.

Example usage, system of 900 water molecules in GROMACS units simulated in
NVT (note that this example leaves some fields in :class:`.SystemData`
empty, as well as the trajectory of some observables and the position and
velocities):
::

   import physical_validation as pv

   parser = pv.data.FlatfileParser()

   system = pv.data.SystemData(
       natoms=900*3,
       nconstraints=900*3,
       ndof_reduction_tra=3,
       ndof_reduction_rot=0
   )

   # We need to specify the units in which the simulation was performed,
   # specifically the value of k_B in the used energy units, the conversion
   # factor of the simulation units to the physical validation units
   # (*_conversion keywords), and a string representation of the simulation
   # units (*_str keywords, used for output only).
   # See documentation below about UnitData object for more details.
   units = pv.data.UnitData(
       kb=8.314462435405199e-3,
       energy_str='kJ/mol',
       energy_conversion=1.0,
       length_str='nm',
       length_conversion=1.0,
       volume_str='nm^3',
       volume_conversion=1.0,
       temperature_str='K',
       temperature_conversion=1.0,
       pressure_str='bar',
       pressure_conversion=1.0,
       time_str='ps',
       time_conversion=1.0
   )

   ensemble = pv.data.EnsembleData(
       ensemble='NVT',
       natoms=900*3,
       volume=3.01125**3,
       temperature=298.15
   )

   res = parser.get_simulation_data(
       units=units, ensemble=ensemble, system=system,
       kinetic_ene_file='kinetic.dat',
       potential_ene_file='potential.dat',
       total_ene_file='total.dat'
   )

Additional examples
-------------------

Use :code:`MDAnalysis` to create mass vector
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Using :code:`MDAnalysis`, creating a mass vector which can be fed to
:attr:`.SystemData.mass` is straightforward. See the following snippet
for an example using a GROMACS topology:
::

   import MDAnalysis as mda
   import numpy as np

   u = mda.Universe('system.gro')
   mass=np.array([u.atoms[i].mass for i in range(len(u.atoms))])

Use :code:`MDAnalysis` to define molecule groups for equipartition testing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:func:`physical_validation.kinetic_energy.equipartition` allows to specify
molecule groups which can be tested for equipartition. The segments used in
:code:`MDAnalysis` can easily be used to define molecule groups as input to
the equipartition check:
::

   import MDAnalysis as mda
   import numpy as np

   u = mda.Universe('system.tpr', 'system.gro')
   molec_groups = []
   for i in range(len(u.segments)):
       seg = u.segments[i]
       molec_groups.append(np.array([seg.atoms[j].index for j in range(len(seg.atoms))]))

Use :code:`MDAnalysis` to read position and velocity trajectory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:code:`MDAnalysis` also makes it easy to create :class:`.TrajectoryData` objects
which require position and velocity trajectories as inputs. Given a
:code:`Universe` object which contains a trajectory, we can simply use a list
comprehension to create a full trajectory in memory:
::

   import MDAnalysis as mda
   import numpy as np
   import physical_validation

   u = mda.Universe('system.tpr', 'system.trr')
   trajectory = physical_validation.data.TrajectoryData(
       position=[frame.positions for frame in u.trajectory],
       velocity=[frame.velocities for frame in u.trajectory])

We can also use the atom selector to only feed part of the trajectory
to the :code:`physical_validation` tests.
This is useful if we want to analyze the equipartition of parts of the
system only (e.g. the solute) which can massively speed up the
validation check. Note that we have to adapt the :class:`.SystemData` object
accordingly to inform :code:`physical_validation` that we are only analyzing
part of the system.
::

   import MDAnalysis as mda
   import numpy as np
   import physical_validation

   u = mda.Universe('system.tpr', 'system.trr')
   protein = u.select_atoms('protein')
   trajectory = physical_validation.data.TrajectoryData(
       position=[protein.positions for _ in u.trajectory],
       velocity=[protein.velocities for _ in u.trajectory])

.. note:: :code:`MDAnalysis` uses Å (ångström) as a length unit. Don't forget to
          choose the :class:`.UnitData` accordingly!


.. _simulationdata_details:

Data contained in :class:`.SimulationData` objects
==================================================

Units: :obj:`.SimulationData.units` of type :class:`.UnitData`
--------------------------------------------------------------
Attributes:

* :attr:`.UnitData.kb`, :code:`float`
* :attr:`.UnitData.energy_conversion`, :code:`float`
* :attr:`.UnitData.length_conversion`, :code:`float`
* :attr:`.UnitData.volume_conversion`, :code:`float`
* :attr:`.UnitData.temperature_conversion`, :code:`float`
* :attr:`.UnitData.pressure_conversion`, :code:`float`
* :attr:`.UnitData.time_conversion`, :code:`float`
* :attr:`.UnitData.energy_str`, :code:`str`
* :attr:`.UnitData.length_str`, :code:`str`
* :attr:`.UnitData.volume_str`, :code:`str`
* :attr:`.UnitData.temperature_str`, :code:`str`
* :attr:`.UnitData.pressure_str`, :code:`str`
* :attr:`.UnitData.time_str`, :code:`str`

The information about units consists of different parts:

* The value of :math:`k_B` in the used energy units,
* the conversion factor to :code:`physical_validation` units (kJ/mol, nm, nm^3, K, bar, ps, the same as the GROMACS default units), and
* the name of the units (:code:`energy_str`, :code:`length_str`, :code:`volume_str`, :code:`temperature_str`, :code:`pressure_str`, :code:`time_str`).

The names are only used for output (console printing and plotting), and are optional.
The conversion factors and kB are, on the other hand, used in computations and need
to be given. To avoid silent errors, these keywords to not have defaults and must be specified. 

Needed by

  * :func:`physical_validation.ensemble.check`
  * :func:`physical_validation.ensemble.estimate_interval`
  * :func:`physical_validation.kinetic_energy.distribution`, only

    - :attr:`.UnitData.kb`

Ensemble: :obj:`.SimulationData.ensemble` of type :class:`.EnsembleData`
------------------------------------------------------------------------
Attributes:

* :attr:`.EnsembleData.ensemble`, :code:`str`
* :attr:`.EnsembleData.natoms`, :code:`int`
* :attr:`.EnsembleData.mu`, :code:`float`
* :attr:`.EnsembleData.volume`, :code:`float`
* :attr:`.EnsembleData.pressure`, :code:`float`
* :attr:`.EnsembleData.energy`, :code:`float`
* :attr:`.EnsembleData.temperature`, :code:`float`

The ensemble is a string indicating the thermodynamical ensemble a simulation was
performed in, and is any of :code:'NVE', :code:'NVT', :code:'NPT', :code:'muVT'.

Depending on the ensemble, :class:`.EnsembleData` then holds additional information defining
the ensemble, such as the number of particles N, the chemical potential mu, the
volume V, the pressure P, the constant energy E or the temperature T. While any
of these additional information are technically optional, most of them are needed by certain
tests, such that not fully defining the ensemble results in warnings. The notable
exception to this rule is the constant energy E for NVE, which is not needed
by any test and can hence be omitted without raising a warning.

Needed by
  * :func:`physical_validation.kinetic_energy.distribution`
  * :func:`physical_validation.ensemble.check`

System: :obj:`.SimulationData.system` of type :class:`.SystemData`
------------------------------------------------------------------
Attributes:

    * :attr:`.SystemData.natoms`, the total number of atoms in the system;
      e.g. for a system containing 100 water molecules: :code:`system_data.natoms = 300`
    * :attr:`.SystemData.nconstraints`, the total number of constraints in the
      system, not including the global translational and rotational constraints
      (see next two attributes); e.g. for a system containing 100 *rigid* water molecules:
      :code:`system_data.nconstraints = 300`
    * :attr:`.SystemData.ndof_reduction_tra`, global reduction of translational
      degrees of freedom (e.g. due to constraining the center of mass of the system)
    * :attr:`.SystemData.ndof_reduction_rot`, global reduction of rotational
      degrees of freedom (e.g. due to constraining the center of mass of the system)
    * :attr:`.SystemData.mass`, a list of the mass of every atom in the system;
      e.g. for a single water molecule: :code:`system_data.mass = [15.9994, 1.008, 1.008]`
    * :attr:`.SystemData.molecule_idx`, a list of the index of the first atom of every
      molecule (this assumes that the atoms are sorted by molecule); e.g. for a system
      containing 3 water molecules: :code:`system_data.molecule_idx = [0, 3, 6]`
    * :attr:`.SystemData.nconstraints_per_molecule`, a list of the number of
      constraints in every molecule; e.g. for a system containing 3 *rigid* water
      molecules: :code:`system_data.nconstraints_per_molecule = [3, 3, 3]`
    * :attr:`.SystemData.bonds`, a list containing all bonds in the system;
      e.g. for a system containing 3 water molecules:
      :code:`system_data.bonds = [[0, 1], [0, 2], [3, 4], [3, 5], [6, 7], [6, 8]]`
    * :attr:`.SystemData.constrained_bonds`, a list containing only the constrained
      bonds in the system, must be a subset of :attr:`.SystemData.bonds` (and equal, if
      all bonds are constrained).

.. todo:: Currently, there is some redundancy in the attributes listed above. The
   :attr:`.SystemData.bonds` and :attr:`.SystemData.constrained_bonds` are
   reserved for future use - included already in the information about the system,
   but not yet used by any tests included in the currently published package. In a
   future version, the :class:`.SystemData` should be streamlined to make the object
   initialization easier.

Needed by

  * :func:`physical_validation.kinetic_energy.distribution`, partially:

    - :attr:`.SystemData.natoms`,
    - :attr:`.SystemData.nconstraints`,
    - :attr:`.SystemData.ndof_reduction_tra`,
    - :attr:`.SystemData.ndof_reduction_rot`

  * :func:`physical_validation.kinetic_energy.equipartition`, all attributes except
    :attr:`.SystemData.bonds` and :attr:`.SystemData.constrained_bonds`.

Observables: :obj:`.SimulationData.observables` of type :class:`.ObservableData`
--------------------------------------------------------------------------------
Attributes:

  * :attr:`.ObservableData.kinetic_energy`, the kinetic energy trajectory (nframes x 1),
    also accessible via :code:`observable_data['kinetic_energy']`
  * :attr:`.ObservableData.potential_energy`, the potential energy trajectory (nframes x 1),
    also accessible via :code:`observable_data['potential_energy']`
  * :attr:`.ObservableData.total_energy`, the total energy trajectory (nframes x 1),
    also accessible via :code:`observable_data['total_energy']`
  * :attr:`.ObservableData.volume`, the volume trajectory (nframes x 1),
    also accessible via :code:`observable_data['volume']`
  * :attr:`.ObservableData.pressure` the pressure trajectory (nframes x 1),
    also accessible via :code:`observable_data['pressure']`
  * :attr:`.ObservableData.temperature` the temperature trajectory (nframes x 1),
    also accessible via :code:`observable_data['temperature']`
  * :attr:`.ObservableData.constant_of_motion` the constant of motion trajectory (nframes x 1),
    also accessible via :code:`observable_data['constant_of_motion']`
  * :attr:`.ObservableData.number_of_species` the trajectory of the number of molecules of a species,
    used for muVT, (nframes x num_species),
    also accessible via :code:`observable_data['number_of_species']`

Needed by

  * :func:`physical_validation.kinetic_energy.distribution`

    - :attr:`.ObservableData.kinetic_energy`

  * :func:`physical_validation.ensemble.check`

    - :attr:`.ObservableData.total_energy`, or
    - :attr:`.ObservableData.potential_energy`,
    - :attr:`.ObservableData.volume` (for NPT),
    - :attr:`.ObservableData.number_of_species` (for muVT)

  * :func:`physical_validation.integrator.convergence`

    - :attr:`.ObservableData.constant_of_motion`

Atom trajectories: :obj:`.SimulationData.trajectory` of type :class:`.TrajectoryData`
-------------------------------------------------------------------------------------
Attributes:

  * :attr:`.TrajectoryData.position`, the position trajectory (nframes x natoms x 3),
    also accessible via :code:`trajectory_data['position']`
  * :attr:`.TrajectoryData.velocity`, the velocity trajectory (nframes x natoms x 3),
    also accessible via :code:`trajectory_data['velocity']`

Needed by

  * :func:`physical_validation.kinetic_energy.equipartition`


Time step: :obj:`.SimulationData.dt` of type :code:`float`
----------------------------------------------------------
The timestep used during the simulation run, a single :code:`float` value.

Needed by

  * :func:`physical_validation.integrator.convergence`
