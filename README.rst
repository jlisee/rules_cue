.. role:: tool(emphasis)

===================
CUE rules for Bazel
===================

.. External links
.. See https://stackoverflow.com/a/4836544/31818 for this abomination:
.. |the cue tool| replace:: the :tool:`cue` tool
.. _the cue tool:
.. _cue: https://cuelang.org/docs/

Integrate |the cue tool|_ into your Bazel projects to define data in CUE
and export it as CUE, JSON, YAML, or plain text. The rules register their
own ``cue`` toolchain, drive the tool through Bazel actions, and support
module-, instance-, and standalone-mode evaluation, with optional
compile-time injection of tagged fields (including Bazel workspace-status
"stamp" values).

.. contents:: Contents
   :depth: 2
   :local:

-----

Overview
========

This repository exposes Starlark rules and macros for invoking
``cue export`` and ``cue def`` from Bazel. The rules come in two families:

- ``cue_exported_*`` — runs ``cue export``, producing a single
  evaluated value in ``cue``, ``json``, ``yaml``, or ``text`` form.
  Suited to emitting data for downstream consumers (Kubernetes manifests,
  configuration bundles, etc.).
- ``cue_consolidated_*`` — runs ``cue def``, producing a consolidated
  representation that preserves schemas, definitions, and constraints in
  the same formats.

Each family has three variants, picked according to how the input files
are organized:

- ``*_standalone_files`` — a bag of CUE (and, optionally, non-CUE) files
  that do not belong to any CUE module. The tool treats each file as a
  "packageless" input.
- ``*_files`` — CUE files that sit inside a CUE module
  (``cue.mod/module.cue``) but that you want to evaluate outside any
  particular package.
- ``*_instance`` — a CUE *instance*: a directory's worth of files that
  constitute a single CUE package, declared by a ``cue_instance`` target.
  Other instances may be referenced as ``deps``.

Supporting rules:

- ``cue_module`` declares a CUE module by pointing at its
  ``cue.mod/module.cue`` file (and, optionally, the contents of
  ``cue.mod/{gen,pkg,usr}``).
- ``cue_instance`` declares a CUE package instance rooted in a module.

-----

Setup (Bzlmod)
==============

These rules target Bzlmod (Bazel 7+). ``rules_cue`` is published to the
`Bazel Central Registry <https://registry.bazel.build/modules/rules_cue>`_.
Add a dependency on it in your ``MODULE.bazel``, then register the
``cue`` toolchain supplied by the module's extension:

.. code-block:: starlark

   bazel_dep(name = "rules_cue", version = "0.17.1")

   cue = use_extension("@rules_cue//cue:extensions.bzl", "cue")
   use_repo(cue, "cue_tool_toolchains")

   register_toolchains("@cue_tool_toolchains//:all")

Consult the registry page linked above for the latest published
version. To track an unreleased commit instead, override the module:

.. code-block:: starlark

   bazel_dep(name = "rules_cue", version = "0.17.1")
   git_override(
       module_name = "rules_cue",
       remote = "https://github.com/seh/rules_cue.git",
       commit = "<commit SHA>",
   )

Load the rules in ``BUILD.bazel`` files from ``@rules_cue//cue:cue.bzl``:

.. code-block:: starlark

   load(
       "@rules_cue//cue:cue.bzl",
       "cue_consolidated_files",
       "cue_consolidated_instance",
       "cue_consolidated_standalone_files",
       "cue_exported_files",
       "cue_exported_instance",
       "cue_exported_standalone_files",
       "cue_instance",
       "cue_module",
   )

-----

Concepts
========

CUE modules
-----------

A CUE *module* is a directory tree rooted at a ``cue.mod`` directory
containing a ``module.cue`` file. Rules whose names end in ``_files`` or
``_instance`` need a ``cue_module`` target that identifies this file, so
that the tool can resolve imports relative to the module root.

CUE instances
-------------

A CUE *instance* is a directory of CUE files that share a package name.
Declare one with ``cue_instance``, identifying its ancestor module and
any other instances it imports. The ``*_instance`` rules evaluate the
package as a whole, honoring import declarations.

Standalone mode
---------------

Files passed to a ``*_standalone_files`` rule are treated as inputs with
no enclosing CUE module or package. This is the simplest mode and the
right default when your inputs are a handful of self-contained CUE files
(possibly combined with JSON, YAML, or raw text to be merged in).

``export`` vs. ``def``
----------------------

``cue export`` evaluates the configuration and emits a single concrete
value (``cue_exported_*``). ``cue def`` emits a consolidated definition
that keeps schemas, definitions, and constraints intact
(``cue_consolidated_*``). Both subcommands perform full CUE
evaluation — meaning either one will fail the build if any constraint
in the inputs is violated.

-----

Rule reference
==============

Rule signatures below use Starlark keyword syntax. Unless marked
"(required)," attributes are optional.

cue_module
----------

.. code-block:: starlark

   cue_module(name = "cue.mod", file = "module.cue", srcs = [])

Declares a CUE module by pointing at its ``module.cue`` file. The
convention is to place this target inside the ``cue.mod`` subdirectory of
the module root and to use the default rule name ``"cue.mod"``.

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Attribute
     - Description
   * - ``name``
     - Target name (defaults to ``"cue.mod"``).
   * - ``file``
     - Label of the module's ``module.cue`` file. Must be named exactly
       ``module.cue`` and sit in a directory named ``cue.mod``
       (defaults to ``"module.cue"``).
   * - ``srcs``
     - CUE files defining external packages from the ``cue.mod``
       ``gen/``, ``pkg/``, and ``usr/`` directories.

cue_instance
------------

.. code-block:: starlark

   cue_instance(
       name,
       ancestor,            # required: a cue_module or dominating cue_instance
       srcs,                # required: CUE files for this package
       deps = [],           # other cue_instance targets this package imports
       directory_of = None, # override the instance directory designator
       package_name = "",   # override the package name (defaults to dir basename)
   )

Declares a CUE instance: a set of files constituting a single CUE
package within the module identified transitively via ``ancestor``.

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Attribute
     - Description
   * - ``ancestor`` (required)
     - The containing ``cue_module`` target, or a dominating
       ``cue_instance`` target. Must provide ``CUEModuleInfo`` or
       ``CUEInstanceInfo``.
   * - ``srcs`` (required)
     - CUE files that are part of this package.
   * - ``deps``
     - ``cue_instance`` targets corresponding to this package's import
       declarations.
   * - ``directory_of``
     - Label of a file (or directory) whose containing directory should
       be used as the instance directory. Defaults to the directory of
       the first file in ``srcs``.
   * - ``package_name``
     - CUE package name for the instance. Defaults to the basename of
       the instance directory.

Common output rule attributes
------------------------------

All output-producing rules (``cue_exported_*`` and
``cue_consolidated_*``) accept the following attributes in addition to
their family-specific ones.

Source inputs
~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Attribute
     - Description
   * - ``srcs``
     - Additional input files that are not part of a CUE package. For
       ``*_standalone_files`` rules this is typically the entire input
       set; for ``*_files`` and ``*_instance`` rules these are extra
       "packageless" inputs merged alongside the package.
   * - ``qualified_srcs``
     - ``label_keyed_string_dict`` of additional input files whose type
       cannot be guessed from the file extension. Each value is a
       qualifier written without the trailing colon (for example,
       ``"yaml"`` or ``"text"``).

Output shaping and injection
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Attribute
     - Description
   * - ``expression``
     - A CUE expression selecting a single value to emit (``cue``'s
       ``--expression``).
   * - ``inject``
     - ``string_dict`` of ``key=value`` pairs injected into tagged
       fields (``cue``'s ``--inject``). Values wrapped in ``{…}`` are
       treated as workspace-status placeholders; see "Stamping" below.
   * - ``inject_shorthand``
     - List of shorthand values injected into tagged fields (for CUE
       tags that take no name).
   * - ``inject_system_variables``
     - Boolean; when true, injects the predefined set of system
       variables as tagged fields (``cue``'s ``--inject-vars``).
   * - ``concatenate_objects``
     - Boolean; concatenate multiple objects into a list
       (``--list``).
   * - ``merge_other_files``
     - Boolean (default ``True``); merge non-CUE files. Set to ``False``
       to disable (``--merge=false``).
   * - ``output_package_name``
     - Name of the CUE package within which to generate CUE output
       (``--package``). (``non_cue_file_package_name`` is accepted as a
       deprecated alias.)
   * - ``path``
     - List of elements of a CUE path at which to place top-level
       values (``--path``). Each element may be a CUE field
       (terminated by ``:`` for a field or ``::`` for a definition) or
       a CUE expression evaluated within the value.
   * - ``with_context``
     - Boolean; evaluate ``path`` elements within a struct that
       identifies source data, file name, record index, and record
       count (``--with-context``).
   * - ``stamping_policy``
     - One of ``"Allow"`` (default), ``"Force"``, or ``"Prevent"``.
       Controls whether ``inject`` values wrapped in ``{…}`` are
       replaced with values from ``stable-status.txt`` /
       ``volatile-status.txt``. ``"Allow"`` stamps only when
       ``bazel build --stamp`` is active; ``"Force"`` stamps
       unconditionally; ``"Prevent"`` disables stamping.

Exported-output attributes (``cue_exported_*``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Attribute
     - Description
   * - ``output_format``
     - One of ``"cue"``, ``"json"`` (default), ``"yaml"``, or
       ``"text"``.
   * - ``escape``
     - Boolean; apply HTML escaping (``--escape``).
   * - ``result``
     - Output file label. Defaults to ``<name>.<ext>`` where the
       extension matches ``output_format`` (``json``, ``yaml``,
       ``cue``, or ``txt``).

Consolidated-output attributes (``cue_consolidated_*``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Attribute
     - Description
   * - ``output_format``
     - One of ``"cue"`` (default), ``"json"``, ``"yaml"``, or
       ``"text"``.
   * - ``inline_imports``
     - Boolean; expand references to non-core imports
       (``--inline-imports``).
   * - ``result``
     - Output file label. Defaults to ``<name>.<ext>`` where the
       extension matches ``output_format``.

cue_exported_standalone_files
-----------------------------

.. code-block:: starlark

   cue_exported_standalone_files(
       name,
       srcs,
       # ... + common output-producing and exported-output attributes
   )

Runs ``cue export`` over a bag of packageless CUE/JSON/YAML/text files.
No ``cue_module`` is involved.

cue_exported_files
------------------

.. code-block:: starlark

   cue_exported_files(
       name,
       srcs,
       module,        # required: a cue_module target
       deps = [],     # cue_instance targets referenced by srcs
       # ... + common output-producing and exported-output attributes
   )

Runs ``cue export`` over files that live inside a CUE module but that
are not themselves an instance. Use it for top-level CUE files that
import instances from the same module.

cue_exported_instance
---------------------

.. code-block:: starlark

   cue_exported_instance(
       name,
       instance,      # required: a cue_instance target
       # ... + common output-producing and exported-output attributes
   )

Runs ``cue export`` over a complete CUE instance (package). The
instance's transitive ``deps`` are supplied automatically through its
``CUEInstanceInfo`` provider.

cue_consolidated_standalone_files
---------------------------------

.. code-block:: starlark

   cue_consolidated_standalone_files(
       name,
       srcs,
       # ... + common output-producing and consolidated-output attributes
   )

Runs ``cue def`` over a bag of packageless files.

cue_consolidated_files
----------------------

.. code-block:: starlark

   cue_consolidated_files(
       name,
       srcs,
       module,        # required: a cue_module target
       deps = [],
       # ... + common output-producing and consolidated-output attributes
   )

Runs ``cue def`` over files inside a CUE module.

cue_consolidated_instance
-------------------------

.. code-block:: starlark

   cue_consolidated_instance(
       name,
       instance,      # required: a cue_instance target
       # ... + common output-producing and consolidated-output attributes
   )

Runs ``cue def`` over a complete CUE instance.

-----

Forcing evaluation to validate CUE files
========================================

``cue_instance`` and ``cue_module`` files do not evaluate the contents of
CUE the files.  To ensure your cue are correct and produce consistent
output you need to use the rules in either a ``cue_exported_*`` or
``cue_consolidated_*`` target. One you do that a ``bazel build`` of the
those rules will validate the CUE files.

When the CUE evaluator rejects the input, Bazel's output looks roughly
like::

   ERROR: .../BUILD.bazel:N:M: CUEExport action for target //pkg:my_rule failed
   some.field: conflicting values "a" and "b"

Tips for CI and validation pipelines
------------------------------------

- Use ``bazel build`` rather than ``bazel test`` to run the evaluator
  for rules that do not register as tests. (You can also wrap the
  outputs in ``diff_test`` targets against golden files for regression
  coverage; see ``test/testdata/...`` for examples.)
- If your rules inject stamped values, remember that
  ``bazel build --stamp`` (or ``stamping_policy = "Force"``) is needed
  to substitute values from ``stable-status.txt`` /
  ``volatile-status.txt``; without stamping, the placeholder strings
  ``{KEY}`` are passed through literally, which may itself trip the
  evaluator.
- The ``cue`` toolchain registered by the module extension is what
  actually performs evaluation. Keep its version pinned via your
  ``MODULE.bazel`` lockfile so that validation is reproducible.

-----

Examples
========

A standalone-mode example from ``examples/bzlmod/root/BUILD.bazel``:

.. code-block:: starlark

   load(
       "@bazel_skylib//rules:write_file.bzl",
       "write_file",
   )
   load(
       "@rules_cue//cue:cue.bzl",
       "cue_exported_standalone_files",
   )

   write_file(
       name = "generated_entries",
       out = "extra-entries.cue",
       content = [
           "package contacts",
           "extra_entries: [{",
           "  name: common: \"Cher\"",
           "  birth: month: \"May\"",
           "  birth: year: 1946",
           "}]",
       ],
   )

   cue_exported_standalone_files(
       name = "root",
       srcs = [
           "entries.cue",
           "schema.cue",
           ":generated_entries",
       ],
       expression = "list.Sort(list.Concat([entries, extra_entries]), {x: {}, y: {}, less: x.name.common < y.name.common})",
   )

A module-mode example from
``test/testdata/hello_world/BUILD.bazel``:

.. code-block:: starlark

   load(
       "//cue:cue.bzl",
       "cue_exported_files",
   )

   _MODULE = "//test/testdata/hello_world/cue.mod"

   cue_exported_files(
       name = "hello_world",
       srcs = ["hello_world.cue"],
       module = _MODULE,
   )

   cue_exported_files(
       name = "message",
       srcs = ["hello_world.cue"],
       expression = "message",
       module = _MODULE,
       output_format = "text",
   )

   cue_exported_files(
       name = "de",
       srcs = ["de.cue"],
       module = _MODULE,
       deps = ["//test/testdata/hello_world/lang:cue_de_library"],
   )

Additional worked examples live under ``test/testdata/`` (covering
injection, stamping, ``path``, consolidated output, and more) and under
``examples/bzlmod/``.

-----

External links
==============

- `CUE documentation <https://cuelang.org/docs/>`_
- `Bazel modules (Bzlmod) <https://bazel.build/external/module>`_
- `Workspace status and stamping <https://bazel.build/docs/user-manual#workspace-status>`_
