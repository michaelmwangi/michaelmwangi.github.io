---
layout: post
title:  "Introduction to Force Fields in Molecular Dynamics"
date:   2017-10-20 
categories: updates
---
{% include mathjax.html %}
One of the applications of computer simulations is Molecular Dynamics (MD), it  allows us to to study the trajectories of particles with respect to time based on integrating Newton's equations of motion. This in the end allows us to study the properties of a chemical system.

For us to carry out an MD simulation we can take on the daunting task of writing the simulation software code or we can use readily available battle tested software e.. Gromacs, NAMD, LAMMPs etc. Most of these softwares are free to download and install on your personal machine or even in computational clusters. Identifying the software to use is definitely the starting point of carrying out your simulation but it’s not sufficient as we need to define an accurate model that represents the interatomic forces acting between particles in the system.

One of the ways of determining the interatomic forces is by use of *ab initio* methods which involve solving the electronic structure for a particular configuration then finally calculating the resulting forces on each atom. Simulations based on these methods are referred to as *ab initio* MD. However we often need simulations whose time scales and the size of the system would be prohibitively expensive if we do the calculations from the first principles method hence we are forced to make some assumptions by making use of empirical force field based Molecular Dynamics.

Force fields are mathematical expressions that describe the interactions between particles in a system. By using force fields we are able to show the dependence of the energy of a system on the position of the particles.

A force field model primarily consists of ;
1. Atoms , represented as point particles in a 3D space ; We get the coordinates of the system.  

 Unlike the *ab initio* method we don’t concern ourselves with the electronic structure.

2. Structural elements such as  bonds, angles, dihedral angles, torsions.  
  

In mathematical terms a force field expression typically takes the form below.

$$E_{total} = E_{intramolecular} + E_{intermolecular}$$

The accuracy of the above energy model is an obvious limitation of force fields and it is where in the bulk of the work lies when developing a new force field.

Our energy equation above needs to quantify certain aspects of the system in order to truly represent a real chemical system. One of the ways of obtaining these constants is through empirical methods where the values are obtained by fitting experimental data such as infrared, Raman spectroscopy, NMR etc. For example in force field based MD, molecules are defined as a set of atoms which are held together by a simple elastic bond whose stretching can be describe by a simple harmonic function, the spring constant can be obtained from infrared or Raman spectra . The spring constant here is what is  referred to as a force field parameter.

## Intramolecular energy terms

These terms describe the energy inside the molecule. This term may be decomposed to;

$$ E_{intramolecular}  = \sum_{bonds}\frac{1}{2}k_b(r-r_o) + \sum_{angles}\frac{1}{2}k_a(\theta-\theta_0)^2+\sum_{torsions}\frac{V_n}{2}[1+cos(n\theta-\delta)]$$

The torsion energy term is required for systems where by the molecule has more than 4 atoms in a row.

When dealing with some particular groups such as $$sp_2$$ hybridized carbons in carbonyl groups or in aromatic rings and additional term is added in-order to capture the out-of-plane motions.

$$E_{improper torsion} = \sum_{improper}\frac{k_imp}{2}[1+cos(2\omega-\pi)]$$


## Intermolecular energy terms

These terms include energy contributions which results from molecule-molecule interactions and include the electrostatic and Van Der Waal's interactions

$$\sum_{intermolecular} = \sum_{electrostatic}\frac{q_1q_2}{4\pi\epsilon_0r}+\sum_{vdw}[(\frac{r_0}{r_ij})^{12} - 2(\frac{r_0}{r_ij})^{6}]$$

In a nutshell these are the core parts of a force field. There are many type of force field that exist in literature for different system and some of them include;

1. AMBER
2. CHARMM
3. OPLS
4. GROMOS

In the next series we will be discussing in detail some of these force fields and their implementation in popular molecular dynamics software.


