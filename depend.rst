=====================
Advanced dependencies
=====================

.. highlight:: bash

.. index:: PYTHON_REQ_USE
.. index:: python_gen_impl_dep

Requesting USE flags on the Python interpreter
==============================================
While the majority of Python standard library modules are available
unconditionally, a few are controlled by USE flags.  For example,
the sqlite3_ module requires ``sqlite`` flag to be enabled
on the interpreter.  If a package requires this module, it needs
to enforce the matching flag via a USE dependency.

In order to create a USE dependency on the Python interpreter, set
``PYTHON_REQ_USE`` before inheriting the eclass.  This will cause
the eclass to generate appropriate dependency string in ``PYTHON_DEPS``.

.. code-block:: bash
   :emphasize-lines: 7

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_6 )
    PYTHON_REQ_USE="sqlite"
    inherit python-r1 gnome2-utils meson xdg-utils

    DESCRIPTION="Modern music player for GNOME"
    HOMEPAGE="https://wiki.gnome.org/Apps/Lollypop"
    SRC_URI="https://adishatz.org/${PN}/${P}.tar.xz"
    KEYWORDS="~amd64"

    LICENSE="GPL-3"
    SLOT="0"
    REQUIRED_USE=${PYTHON_REQUIRED_USE}

    DEPEND="${PYTHON_DEPS}
        ..."

Full USE dependency syntax is permitted.  For example, you can make
the dependency conditional to a flag on the package:

.. code-block:: bash
   :emphasize-lines: 7,17

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6

    PYTHON_COMPAT=( python3_6 )
    PYTHON_REQ_USE="sqlite?"
    inherit distutils-r1

    DESCRIPTION="A lightweight password-manager with multiple database backends"
    HOMEPAGE="https://pwman3.github.io/pwman3/"
    SRC_URI="https://github.com/pwman3/pwman3/archive/v${PV}.tar.gz -> ${P}.tar.gz"

    LICENSE="GPL-3"
    SLOT="0"
    KEYWORDS="~amd64"
    IUSE="mongodb mysql postgres +sqlite"

Finally, there are cases when the problem cannot be fully solved using
a single USE dependency.  Additional Python interpreter dependencies
with specific USE flags can be constructed using ``python_gen_impl_dep``
helper then.  For example, the following ebuild requires Python with
SQLite support when running tests:

.. code-block:: bash
   :emphasize-lines: 24

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7
    PYTHON_COMPAT=( python{2_7,3_{6,7,8}} pypy3 )

    inherit distutils-r1

    DESCRIPTION="Let your Python tests travel through time"
    HOMEPAGE="https://github.com/spulec/freezegun"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="Apache-2.0"
    SLOT="0"
    KEYWORDS="~alpha ~amd64 ~arm ~arm64 ~hppa ~ia64 ~m68k ~mips ~ppc ~ppc64 ~s390 ~sh ~sparc ~x86 ~amd64-linux ~x86-linux ~ppc-macos ~x64-macos ~x86-macos"

    RDEPEND="
        >dev-python/python-dateutil-2.0[${PYTHON_USEDEP}]
        dev-python/six[${PYTHON_USEDEP}]
    "
    DEPEND="${RDEPEND}
        dev-python/setuptools[${PYTHON_USEDEP}]
        test? (
            $(python_gen_impl_dep sqlite)
            dev-python/mock[${PYTHON_USEDEP}]
            dev-python/nose[${PYTHON_USEDEP}]
        )
    "

    distutils_enable_tests pytest


.. index:: python_gen_cond_dep; for conditional deps

Dependencies conditional to Python version
==========================================
When packaging software for multiple Python versions, it is quite likely
that you'll find yourself needing some packages only with some
of the versions, and not with others.  This is the case with backports
and other compatibility packages.  It also happens if some
of the optional dependencies do not support the full set
of implementations your package supports.

A dependency that applies only to a subset of ``PYTHON_COMPAT`` can
be created using ``python_gen_cond_dep`` function (the same as used
in ``python-single-r1``).  It takes a dependency string template,
followed by zero or more implementation arguments.  The dependencies
are output for every matching implementation.

The dependency template should contain literal (usually escaped through
use of single quotes) ``${PYTHON_USEDEP}`` that will be substituted
with partial USE dependency by the eclass function (when using
``python-single-r1``, ``${PYTHON_SINGLE_USEDEP}`` is also permitted).

The implementation arguments can be:

1. Literal implementation names.  For example, if a particular feature
   is only available on a subset of Python implementations supported
   by the package::

       RDEPEND="
           cli? (
               $(python_gen_cond_dep '
                   dev-python/black[${PYTHON_USEDEP}]
                   dev-python/click[${PYTHON_USEDEP}]
               ' python3_{8..10})
           )
       "

2. ``fnmatch(3)``-style wildcard against implementation names.
   For example, CFFI is part of PyPy's stdlib, so the explicit package
   needs to be only installed for CPython::

       RDEPEND="
           $(python_gen_cond_dep '
               dev-python/cffi[${PYTHON_USEDEP}]
           ' 'python*')
       "

   Remember that the patterns need to be escaped to prevent filename
   expansion from happening.

3. Python standard library versions that are expanded into appropriate
   implementations by the eclass.  For example, this makes it convenient
   to depend on backports::

       RDEPEND="
           $(python_gen_cond_dep '
               dev-python/backports-zoneinfo[${PYTHON_USEDEP}]
           ' 3.8)
       "

   The advantage of this form is that the dependencies automatically
   adjust as we switch PyPy3 to a newer stdlib version.

An important feature of ``python_gen_cond_dep`` is that it handles
removal of old implementations gracefully.  When one of the listed
implementations is no longer supported, it silently ignores it.  This
makes it possible to remove old implementations without having to update
all dependency strings immediately.

For example, in the following example the dependency became empty when
Python 3.7 was removed::

    RDEPEND="
        $(python_gen_cond_dep '
            dev-python/importlib_metadata[${PYTHON_USEDEP}]
        ' python3_7)"


.. index:: cffi, greenlet

Dependencies on CFFI and greenlet
=================================
The PyPy distribution includes special versions of the cffi_
and greenlet_ packages.  For this reason, packages using CFFI
and/or greenlet and supporting PyPy3 need to make the explicit
dependencies conditional to CPython::

    RDEPEND="
        $(python_gen_cond_dep '
            >=dev-python/cffi-1.1.0:=[${PYTHON_USEDEP}]
        ' 'python*')
    "


.. _sqlite3: https://docs.python.org/3.8/library/sqlite3.html
.. _cffi: https://pypi.org/project/cffi/
.. _greenlet: https://pypi.org/project/greenlet/
