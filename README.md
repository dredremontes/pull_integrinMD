# pull_integrinMD
 Steered molecular dynamics simulation of alpha5beta1 integrin and fibronectin

Downloading Gromacs
===========
This implementation was run in Ubuntu 20.04. The molecular dynamics software used was Gromacs 2020.4, which can be downloaded from https://ftp.gromacs.org/pub/gromacs-2020.4.tar.gz and is also available in the ./pull_integrinMD. The source code can be downloaded from https://manual.gromacs/org/2020.4/download.html. The tarball can be unpacked using:

$> tar -xvzf gromacs-2020.4.tar.gz

Please refer to the gromacs installation guide found in https://manual.gromacs.org/2020.4/install_guide/index.html. Check for Clang by running:

$> clang –version 

to check if Clang is already installed. If not, it can be installed by following the instructions here: https://www.ics.uci.edu/~pattis/common/handouts/macclion/clang.html. Next, download cmake from https://cmake.org/download/. 

First, we’ll move the gromacs-2020.4 folder from its current location to the folder where our programs are kept. For my PC, this folder is "C:/Program Files." For MacOS, this will be the Applications folder. Next, we’re going to open up the terminal and navigate to the folder containing gromacs-2020.4. Now we can follow the "Quick and dirty installation" steps from the GROMACS website:

$> cd gromacs-2020.4
$> mkdir build
$> cd build
$> cmake .. -DGMX_BUILD_OWN_FFTW=ON -DREGRESSIONTEST_DOWNLOAD=ON
$> make
$> make check
$> sudo make install
$> source /usr/local/gromacs/bin/GMXRC

Setting up the simulation
==========

All input files required to run the simulations are in ./pull_integrinMD/input/

Starting from scratch, download 7NWL.pdb from https://www.rcsb.org/structure/7NWL and remove the TS2/16 Fv-clasp in PYMOL, VMD, or manually. The file ./pull_integrinMD/input/7NWL_abc has the clasp removed already. Convert the pdb file to a gro file:

$> gmx pdb2gmx -f 7NWL_abc.pdb -o 7NWL_abc.gro

Select AMBER99SB-ildn force field and TIP3P. Rotate the molecule, add a box, solvate, and add ions with:

$> gmx editconf -f 7NWL_abc.gro -rotate 0 0 45 -o 7NWL_abc.gro
$> gmx editconf -f 7NWL_abc.gro -o box.gro -box 18 45 19 -center 9 13 10
$> gmx solvate -cp box.gro -cs spc216.gro -o solv.gro -p topol.top
$> gmx grompp -f ions.mdp -c solv.gro -p topol.top -o ions.tpr
$> gmx genion -s ions.tpr -o solv_ions.gro -p topol.top -pname NA -nname CL -neutral -conc 0.15

Select the SOL option.

Minimization
==========

NOTE: we DO NOT recommend running the minimization, equilibration, and steered locally on a laptop as the computational cost is very high and a lot of data is generated. A supercomputing cluster would be most ideal. These instructions are here FOR REFERENCE ONLY. The results of the simulations can be found in their corresponding folders. However, the trajectory files are not included due to their very large size (> 1 GB)

$> gmx grompp -f minim.mdp -c solv_ions.gro -p topol.top -o em.tpr
$> gmx mdrun -v -deffnm em

Energy can be extracted to an xvg file using:

$> gmx energy -f em.edr -s em.tpr -o em.xvg

Equilibration
==========

NVT:
$> gmx grompp -f nvt.mdp -c em.gro -p topol.top -r em.gro -o nvt.tpr
$> gmx mdrun -deffnm nvt

NPT:
$> gmx grompp -f npt.mdp -c nvt.gro -p topol.top -r nvt.gro -t nvt.cpt -o npt.tpr
$> gmx mdrun -deffnm npt

The RMSD can be calculated with:
$> gmx rms -s npt.tpr -f npt.trr -o rmsd.xvg

The energy, temperature, and pressure can be extracted with:
$> gmx energy -f npt.edr -s npt.tpr -o output.xvg

Steered MD
==========

Define pull groups on intergrin and fibronectin:

$> gmx make_ndx -f npt.gro
$> a8254-8275
$> name 20 int_a
$> a9418-9432
$> name 21 int_b
$> a15796-15811
$> name 22 fn

Create constraints

$> gmx genrestr -f npt.gro -n index.ndx -o R_A.itp
$> gmx genrestr -f npt.gro -n index.ndx -o R_B.itp

These position restraints need to be defined in their respective protein chain .itp files. For the restraint on int_a, we can add the following to the bottom of the topol_Protein_chain_A.itp file:

#ifdef POSRES_R_A
#include "R_A.itp"
#endif

For the restraint on int_b, we can add the following to the bottom of the topol_Protein_chain_B.itp file:
#ifdef POSRES_R_B
#include "R_B.itp"
#endif

We must open up the R_A.itp and R_B.itp files and manually renumber the left-most column, i. This column will contain the original atom numbers, which confuses GROMACS because position restraint (.itp) files always start with the number 1. So we have to go in and manually change ALL the atom numbers in R_A.itp and R_B.itp to 1, 2, 3 and so on. 

Add the following to the pull.mdp, which can be any pull_.mdp file:

define = -DPOSRES_R_A -DPOSRES_R_B

Running the steered MD. Replace :
$> gmx grompp -f pull.mdp -c npt.gro -p topol.top -r npt.gro -n index.ndx -t npt.cpt -o pull.tpr
$> gmx mdrun -deffnm pull -s pull.tpr -pf pullf.xvg -px pullx.xvg

Post-processing:
===============

After running the simulations, we can extract the COM coordinates of the pull groups to calculate extension:

$> gmx trajectory -f sim_name.xtc -s sim_name.tpr -n index.ndx -ox pull-coords.xvg -seltype res_com -y yes

Select the three pull groups defined earlier (int_a, int_b, and fn) to complete the selection and extraction. Gromacs will output a sim-namef.xvg file with the forces. Python can parse through the data files and after some manipulation, we can plot the force vs extension. Gromacs can also condense the trajectory files for faster viewing in VMD by extracting only the protein:

$> gmx trjconv -f sim-name.xtc -s sim-name.tpr -o new-traj.xtc

Then selecting option 1 to select the protein. We can then use VMD to load npt.gro and sim-name.xtc and create movies.