scf --- Mean-field methods
**************************

.. module:: scf
   :synopsis: restricted and unrestricted, closed shell and open shell Hartree-Fock methods

Introduction
============

Caching two-electron integrals
------------------------------

When memory is enough (specified by the :attr:`max_memory` of SCF object), the
SCF object generates all two-electron integrals in memory and cache them in
:attr:`_eri` array.  The default :attr:`max_memory` (defined in
:data:`lib.parameters.MAX_MEMORY`, see :ref:`max_mem`) is 4 GB.  It roughly
corresponds to two-electron real integrals for 250 orbitals.  For small systems,
the cached integrals usually provide the best performance.  If you have enough
main memory in your computer, you can increase the :attr:`max_memory` of SCF
object to cache the integrals in memory.

The cached integrals :attr:`_eri` are treated as a dense tensor.  When system
becomes larger and the two-electron integral tensor becomes sparse, caching
integrals may lose performance advantage.  This is mainly due to the fact that
the implementation of J/K build for the cached integrals did not utilize the
sparsity of the integral tensor.  Also, the data locality was not considered in
the implementation which sometimes leads to bad OpenMP multi-threading speed up.
For large system, the AO-driven direct SCF method is more favorable.


.. _customize_h:

Customizing Hamiltonian
-----------------------

This integral object :attr:`_eri` is not merely used by the mean-field
calculation.  Along with the :meth:`get_hcore` method, this two-electron
integral object will be treated as the Hamiltonian in the post-SCF code whenever
possible.  This mechanism provides a way to model arbitrary fermion system in
PySCF.  You can customize a system by changing the 1-electron Hamiltonian and
the mean-field :attr:`_eri` attribute.  For example, the following code solves a
model system::

    import numpy
    from pyscf import gto, scf, ao2mo, ccsd
    mol = gto.M()
    n = 10
    mol.nelectron = 10
    mf = scf.RHF(mol)
    
    t = numpy.zeros((n,n))
    for i in range(n-1):
        t[i,i+1] = t[i+1,i] = 1.0
    t[n-1,0] = t[0,n-1] = -1.0
    eri = numpy.zeros((n,n,n,n))
    for i in range(n):
        eri[i,i,i,i] = 4.0
    
    mf.get_hcore = lambda *args: t
    mf.get_ovlp = lambda *args: numpy.eye(n)
    # ao2mo.restore(8, eri, n) to get 8-fold symmetry of the integrals
    # ._eri only supports the 2-electron integrals in 4-fold or 8-fold symmetry.
    mf._eri = ao2mo.restore(8, eri, n)
    
    mf.kernel()

    mycc = ccsd.RCCSD(mf).run()
    e,v = mycc.ipccsd(nroots=3)
    print('IP = ', e)
    e,v = mycc.eaccsd(nroots=3)
    print('EA = ', e)

Some post-SCF methods require the 4-index MO integrals.  Depending the available
memory (affected by the value of :attr:`max_memory` in each class), these
methods may not use the "AO integrals" cached in :attr:`_eri`.  To ensure the
post mean-field methods to use the :attr:`_eri` integrals no matter whether the
actual memory usage is over the :attr:`max_memory` limite, you can set the flag
:attr:`incore_anyway` in :class:`Mole` class to ``True`` before calling the
:meth:`kernel` function of the post-SCF methods.  In the following example,
without setting ``incore_anyway=True``, the CCSD calculations will crash::

    import numpy
    from pyscf import gto, scf, ao2mo, ccsd
    mol = gto.M()
    n = 10
    mol.nelectron = n
    mol.max_memory = 0
    mf = scf.RHF(mol)
    
    t = numpy.zeros((n,n))
    for i in range(n-1):
        t[i,i+1] = t[i+1,i] = 1.0
    t[n-1,0] = t[0,n-1] = -1.0
    eri = numpy.zeros((n,n,n,n))
    for i in range(n):
        eri[i,i,i,i] = 4.0
    
    mf.get_hcore = lambda *args: t
    mf.get_ovlp = lambda *args: numpy.eye(n)
    mf._eri = ao2mo.restore(8, eri, n)
    mf.kernel()

    mol.incore_anyway = True
    mycc = ccsd.RCCSD(mf).run()
    e,v = mycc.ipccsd(nroots=3)
    print('IP = ', e)
    e,v = mycc.eaccsd(nroots=3)
    print('EA = ', e)

Holding the entire two-particle interactions matrix elements in memory often
leads to high memory usage.  In the SCF calculation, the memory usage can be
optimized if :attr:`_eri` is sparse.  The SCF iterations requires only the Fock
matrix which in turn calls the J/K build function :meth:`SCF.get_jk` to compute
the Coulomb and HF-exchange matrix.  Overwriting the :meth:`SCF.get_jk` function
can reduce the memory footprint of the SCF part in the above example::

    import numpy
    from pyscf import gto, scf, ao2mo, ccsd
    mol = gto.M()
    n = 10
    mol.nelectron = n
    mol.max_memory = 0
    mf = scf.RHF(mol)
    
    t = numpy.zeros((n,n))
    for i in range(n-1):
        t[i,i+1] = t[i+1,i] = 1.0
    t[n-1,0] = t[0,n-1] = -1.0
    
    mf.get_hcore = lambda *args: t
    mf.get_ovlp = lambda *args: numpy.eye(n)
    def get_jk(mol, dm, *args):
        j = numpy.diag(dm.diagonal()) * 4.
        k = numpy.diag(dm.diagonal()) * 4.
        return j, k
    mf.get_jk = get_jk
    mf.kernel()

Another way to handle the two-particle interactions of large model system is to
use the density fitting/Cholesky decomposed integrals.  See also
:ref:`sl_cderi`.


Program reference
=================

.. automodule:: pyscf.scf
 

Non-relativistic Hartree-Fock
-----------------------------

.. autoclass:: pyscf.scf.hf.SCF
   :members:

----------

.. autoclass:: pyscf.scf.hf.RHF
   :members:

----------

.. autoclass:: pyscf.scf.rohf.ROHF
   :members:

----------

.. autoclass:: pyscf.scf.uhf.UHF
   :members:

----------

.. automodule:: pyscf.scf.hf
   :members:

----------

.. automodule:: pyscf.scf.uhf
   :members:

----------

.. automodule:: pyscf.scf.hf_symm
   :members:

----------

.. automodule:: pyscf.scf.uhf_symm
   :members:


Relativistic Hartree-Fock
-------------------------

.. automodule:: pyscf.scf.dhf
   :members:


addons
------

.. automodule:: pyscf.scf.addons
   :members:

----------

.. automodule:: pyscf.scf.chkfile
   :members:

----------

.. automodule:: pyscf.scf.diis
   :members:

