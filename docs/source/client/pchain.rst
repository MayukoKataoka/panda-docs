=========================
Running workflow
=========================

A workflow is a set of tasks whose relationship is described with a directed acyclic graph (DAG).
In a DAG, a parent task can be connected to one or more child tasks with each edge directed from
the parent task to a child task. A child task processes the output data of the parent task.
It is possible to configure each child task to get started when the parent task produces the entire output data
or partially produces the output data, depending on the use-case.
If a child task is configured as the latter both parent and child tasks
will run in parallel for a while, which will reduce the total execution time of the workflow.
Currently tasks have to be PanDA tasks, but future versions will support more backend systems such as local
batch systems, production system, kubernetes-based resources,
and other workflow management systems, to run some tasks very quickly or outsource sub-workflows.

The user describes a workflow using a yaml-based language, Common Workflow Language
(`CWL <https://www.commonwl.org/user_guide/>`_),
or a Python-based language, `Snakemake <https://snakemake.readthedocs.io/en/stable/>`_,
and submits it to PanDA using ``pchain``.
This page explains how to use ``pchain`` as well as how to describe workflows.

:orange:`Remark: It could be a bit cumbersome and error-prone to edit CWL or snakemake files. We are developing
a handy tool/interface that translates graphical diagrams to workflow descriptions written in
python or yaml on behalf of users. However, you have to play with CWL or snakemake files for now.`


.. contents:: Table of Contents
    :local:

-----------

|br|

Workflow examples with CWL
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Simple task chain
======================

The following cwl code shows a parent-child chain of two prun tasks.

.. figure:: images/pchain_dag_simple.png

.. literalinclude:: cwl/test.cwl
    :language: yaml
    :caption: simple_chain.cwl

The ``class`` field must be :brown:`Workflow` to indicate this code describes a workflow.
There are two prun tasks in the workflow and defined as :blue:`top` and :blue:`bottom` steps in ``steps`` section.
The ``inputs`` section is empty since the workflow doesn't take any input data.
The ``outputs`` section describes the output parameter of the workflow, and it takes only one string type parameter
with an arbitrary name.
The ``outputSource`` connects the output parameter of the *bottom* step to the workflow output parameter.

In the ``steps`` section, each step represents a task with an arbitrary task name, such as :blue:`top`
and :blue:`bottom`.
The ``run`` filed of a prun task is :brown:`prun`. The ``in`` section specifies a set of parameters
correspond to command-line options of prun.

Here is a list of parameters in the ``in`` section to run a prun task.

.. list-table::
   :header-rows: 1

   * - Parameter
     - Corresponding prun option
   * - opt_inDS
     - ---inDS (string)
   * - opt_inDsType
     - No correspondence. Type of inDS (string)
   * - opt_secondaryDSs
     - ---secondaryDSs (a list of strings)
   * - opt_secondaryDsTypes
     - No correspondence. Types of secondaryDSs (a string array)
   * - opt_exec
     - ---exec (string)
   * - opt_useAthenaPackages
     - ---useAthenaPackages (bool)
   * - opt_containerImage
     - ---containerImage (string)
   * - opt_args
     - all other prun options except for listed above (string)

All options ``opt_xyz`` except ``opt_args`` and ``opt_xyzDsTypes`` can be mapped to :hblue:`---xyz` of prun.
``opt_args`` specifies all other prun options such as :hblue:`---outputs`, :hblue:`---nFilesPerJob`,
and :hblue:`---nJobs`.
Essentially,

.. code-block:: yaml

    run: prun
    in:
      opt_exec:
        default: "echo %RNDM:10 > seed.txt"
      opt_args:
        default: "--outputs seed.txt --nJobs 3"

corresponds to

.. code-block:: bash

  prun --exec "echo %RNDM:10 > seed.txt" --outputs seed.txt --nJobs 3

:hblue:`%IN` in ``opt_args`` is expanded to a list of filenames in the input dataset specified in ``opt_inDS``.
It is possible to use :hblue:`%{DSn}` in ``opt_args`` as a placeholder for the input dataset name.

The ``out`` section specifies the task output with an arbitrary string surrendered by brackets.
Note that it is always a single string even if the task produces multiple outputs.
The output of the :blue:`top` task is passed to ``opt_inDS`` of the :blue:`bottom` task.
The :blue:`bottom` task starts processing once the *top* task produces enough output data,
waits if all data currently available has been processed but the :blue:`top` task is still running,
and finishes once all data from the :blue:`top` task is processed.

The user can submit the workflow to PanDA using ``pchain`` that is included in panda-client.
First, create a file called :brown:`simple_chain.cwl` containing the cwl code above.
Next, you need to create an empty yaml file since cwl files work with yaml files that describe workflow inputs.
This example doesn't take an input, so the yaml file can be empty.

.. tabs::

   .. code-tab:: bash ATLAS users

      $ touch dummy.yaml
      $ pchain --cwl simple_chain.cwl --yaml dummy.yaml --outDS user.<your_nickname>.blah

   .. code-tab:: bash DOMA users

      $ touch dummy.yaml
      $ pchain --cwl simple_chain.cwl --yaml dummy.yaml --outDS user.<your_nickname>.blah \
         --vo wlcg --prodSourceLabel test --workingGroup ${PANDA_AUTH_VO}

``pchain`` automatically sends local \*.cwl, \*.yaml, and \*.json files to PanDA together with the workflow.
``--outDS`` is the basename of the datasets for output and log files. Once the workflow is submitted,
the cwl and yaml files are parsed on the server side to generate tasks
with sequential numbers in the workflow. The system uses a combination of the sequential number
and the task name, such as :brown:`000_top` and :brown:`001_bottom`, as a unique identifier for each task.
The actual output dataset name is a combination of ``--outDS``, the unique identifier, and :hblue:`---outputs`
in ``opt_args``. For example, the output dataset name of the :blue:`top` task is
:brown:`user.<your_nickname>.blah_000_top_seed.txt`
and that of the :blue:`bottom` is :brown:`user.<your_nickname>.blah_001_bottom_results.root`.
If :hblue:`---outputs` is a comma-separate
output list, one dataset is created for each output type.

To see all options of ``pchain``

.. prompt:: bash

  pchain --helpGroup ALL

|br|

More complicated chain
========================

The following cwl example describes more complicated chain as shown in the picture below.

.. figure:: images/pchain_dag_combine.png

.. literalinclude:: cwl/sig_bg_comb.cwl
    :language: yaml
    :caption: sig_bg_comb.cwl

The workflow takes two inputs, :blue:`signal` and :blue:`background`. The :blue:`signal` is used as input for
the :blue:`make_signal`
task, while the :blue:`background` is used as input for the :blue:`make_background_1` and
:blue:`make_background_2` tasks.
The :blue:`make_signal` task runs in the busybox container as specified in ``opt_containerImage``, to produce two
types of output data, abc.dat and def.zip, as specified in ``opt_args``.
If the parent task produces multiple types of output data and the child task uses some of them,
their types need to be specified in ``opt_inDsType``.
The :blue:`premix` task takes def.zip from the :blue:`make_signal` task and xyz.pool
from the :blue:`make_background_1` task.


Output data of parent tasks can be passed to a child task as secondary inputs. In this case, they are
specified in ``opt_secondaryDSs`` and their types are specified in ``opt_secondaryDsTypes``.
Note that the stream name, the number of files per job, etc, for each secondary input are specified
using :hblue:`---secondaryDSs` in ``opt_args`` where :hblue:`%{SECDSn}` can
be used as a placeholder for the n-th secondary dataset name.
``MultipleInputFeatureRequirement`` is required if ``opt_secondaryDsTypes`` take multiple input data.

The workflow inputs are described in a yaml file. E.g.,

.. prompt:: bash $ auto

  $ cat inputs.yaml

  signal: mc16_valid:mc16_valid.900248.PG_singlepion_flatPt2to50.simul.HITS.e8312_s3238_tid26378578_00
  background: mc16_5TeV.361238.Pythia8EvtGen_A3NNPDF23LO_minbias_inelastic_low.merge.HITS.e6446_s3238_s3250/

Then submit the workflow.

.. prompt:: bash

  pchain --cwl sig_bg_comb.cwl --yaml inputs.yaml --outDS user.<your_nickname>.blah

If you need to run the workflow with different input data it enough to submit it with a different yaml file.

|br|

Sub-workflow and parallel execution with scatter
======================================================

A workflow can be used as a step in another workflow.
The following cwl example uses the above :brown:`sig_bg_comb.cwl` in the :blue:`many_sig_bg_comb` step.

.. figure:: images/pchain_dag_scatter.png

.. literalinclude:: cwl/merge_many.cwl
    :language: yaml
    :caption: merge_many.cwl

Note that sub-workflows require ``SubworkflowFeatureRequirement``.

It is possible to run a task or sub-workflow multiple times over a list of inputs using
``ScatterFeatureRequirement``.
A popular use-case is to perform the same analysis step on different samples in a single workflow.
The step takes the input(s) as an array and will run on each element of the array as if it were a single input.
The :blue:`many_sig_bg_comb` step above takes two string arrays, :blue:`signals` and :blue:`backgrounds`,
and specifies in the ``scatter`` field that it loops over those arrays.
Output data from all :blue:`many_sig_bg_comb` tasks are fed into the :blue:`merge` task to produce the final output.

The workflow inputs are string arrays like

.. prompt:: bash $ auto

  $ cat inputs2.yaml

  signals:
    - mc16_valid:mc16_valid.900248.PG_singlepion_flatPt2to50.simul.HITS.e8312_s3238_tid26378578_00
    - valid1.427080.Pythia8EvtGen_A14NNPDF23LO_flatpT_Zprime.simul.HITS.e5362_s3718_tid26356243_00

  background:
    - mc16_5TeV.361238.Pythia8EvtGen_A3NNPDF23LO_minbias_inelastic_low.merge.HITS.e6446_s3238_s3250/
    - mc16_5TeV:mc16_5TeV.361239.Pythia8EvtGen_A3NNPDF23LO_minbias_inelastic_high.merge.HITS.e6446_s3238_s3250/

Then submit the workflow.

.. prompt:: bash

  pchain --cwl merge_many.cwl --yaml inputs2.yaml --outDS user.<your_nickname>.blah

|br|

Using Athena
======================

One or more tasks in a single workflow can use Athena as shown in the example below.

.. literalinclude:: cwl/athena.cwl
    :language: yaml
    :caption: athena.cwl

``opt_useAthenaPackages`` corresponds to ``--useAthenaPackages`` of prun to remotely setup Athena with your
locally-built packages.
You can use a different Athena version by specifying :hblue:`---athenaTag` in ``opt_args``.

To submit the task, first you need to setup Athena on local computer, and execute ``pchain``
with ``--useAthenaPackages`` that automatically collect various Athena-related information
from environment variables and uploads a sandbox file from your locally-built packages.

.. prompt:: bash

  pchain --cwl athena.cwl --yaml inputs.yaml --outDS user.<your_nickname>.blah --useAthenaPackages

|br|

Conditional workflow
========================

Workflows can contain conditional steps executed based on their input. This allows workflows
to wait execution of subsequent tasks until previous tasks are done, and
to skip subsequent tasks based on results of previous tasks.
The following example contains conditional branching based on the result of the first step.
Note that this workflows conditional branching require ``InlineJavascriptRequirement`` and CWL version 1.2 or higher.

.. figure:: images/pchain_dag_cond.png

.. literalinclude:: cwl/cond.cwl
    :language: yaml
    :caption: cond.cwl

Both :blue:`bottom_OK` and :blue:`bottom_NG` steps take output data of the :blue:`top` step as input.
The new property ``when`` specifies the condition validation expression that is interpreted by JavaScript.
:hblue:`self.blah` in the expression represents the input parameter :brown:`blah` of the step that is connected
to output data of the parent step. If the parent step is successful :hblue:`self.blah` gives :brown:`True`
while :hblue:`!self.blah` gives :brown:`False`. It is possible to create more complicated expressions using
logical operators (:brown:`&&` for AND and :brown:`||` for OR) and parentheses. The step is executed when the
whole expression gives :brown:`True`.

The :blue:`bottom_NG` step is executed when the :blue:`top` step fails and :hblue:`$(!self.opt_inDS)` gives
:brown:`True`. Note that in this case output data from the :blue:`top` step is empty and
the prun task in the :blue:`bottom_NG` step is executed without ``--inDS``.

|br|

Involving hyperparameter optimization
==================================================

It is possible to run Hyperparameter Optimization (HPO) and chain it with other tasks in the workflow.
The following example shows a chain of HPO and prun tasks.

.. literalinclude:: cwl/hpo.cwl
    :language: yaml
    :caption: hpo.cwl

where the output data of the :blue:`pre_proc` step is used as the training data for the :blue:`main_hpo` step,
and the output data :brown:`metrics.tgz` of the :blue:`main_hpo` step is used as the input for the
:blue:`post_proc` step.
Both :blue:`main_hpo` and :blue:`post_proc` steps specify ``when`` since they waits until the upstream step is done.

The ``run`` filed of a phpo task is :brown:`phpo`.
Here is a list of parameters in the ``in`` section to run a prun task.

.. list-table::
   :header-rows: 1

   * - Parameter
     - Corresponding phpo option
   * - opt_trainingDS
     - ---trainingDS (string)
   * - opt_trainingDsType
     - No correspondence. Type of trainingDS (string)
   * - opt_args
     - all other phpo options except for listed above (string)

``opt_trainingDS`` can be omitted if the HPO task doesn't take a training dataset.
Note that you can put most phpo options in a json and specify the json filename in :hblue:`---loadJson`
in ``opt_args``, rather than constructing a complicated string in ``opt_args``.

.. prompt:: bash $ auto

  $ cat config.json
  {
    "evaluationContainer": "docker://gitlab-registry.cern.ch/zhangruihpc/evaluationcontainer:mlflow",
    "evaluationExec": "bash ./exec_in_container.sh",
    "evaluationMetrics": "metrics.tgz",
    "steeringExec": "run --rm -v \"$(pwd)\":/HPOiDDS gitlab-registry.cern.ch/zhangruihpc/steeringcontainer:0.0.1 /bin/bash -c \"hpogrid generate --n_point=%NUM_POINTS --max_point=%MAX_POINTS --infile=/HPOiDDS/%IN  --outfile=/HPOiDDS/%OUT -l nevergrad\""
  }

  $ pchain -cwl hpo.cwl --yaml dummy.yaml --outDS user.<your_nickname>.blah

|br|

Loops in workflows
====================

Users can have loops in their workflows. Each loop is represented as a sub-workflow with a parameter dictionary.
All tasks in the sub-workflow share the dictionary to generate actual steps. There is a special type of tasks, called
:brown:`junction`, which read outputs from upstream tasks, and produce json files to update the parameter dictionary
and/or make a decision to exit from the sub-workflow.
The sub-workflow is iterated until one of :brown:`junctions` decides to exit. The new iteration is executed
with the updated values in the parameter dictionary, so that each iteration can bring different results.

The ``run`` filed of a junction is :brown:`junction`.
Essentially, a junction is a simplified prun task that processes all input files in a single job
to produce a json file, so there are only a few parameters in the ``in`` section of a junction, as shown below.

.. list-table::
   :header-rows: 1

   * - Parameter
     - Corresponding prun option
   * - opt_inDS
     - Input datasets (a list of strings)
   * - opt_inDsType
     - Types of input datasets (a list of strings. optional)
   * - opt_exec
     - The execution string
   * - opt_containerImage
     - Container image name (string. optional)
   * - opt_args
     - To define additional output files in --outputs

For example, the following pseudo-code snippet has a single loop

.. code-block:: python

  out1 = work_start()
  xxx = 123
  yyy = 0.456
  while True:
      out2 = inner_work_top(out1, xxx)
      out3 = inner_work_bottom(out2, yyy)
      xxx = out2 + 1
      yyy *= out3
      if yyy > xxx:
          break
  out4 = work_end(out3)

The code is described using CWL as follows.

.. figure:: images/pchain_dag_loop.png

.. literalinclude:: cwl/loop.cwl
    :language: yaml
    :caption: loop.cwl

The :blue:`work_loop` step describes the looping stuff in the :hblue:`while` block of the pseudo-code snippet.
It runs a separate CWL file :brown:`loop_body.cwl` and
has :hblue:`loop` in the ``hints`` section to iterate.

.. literalinclude:: cwl/loop_body.cwl
    :language: yaml
    :caption: loop_body.cwl

The local variables in the loop like :brown:`xxx` and :brown:`yyy` are defined in the ``inputs`` section
of :brown:`loop_body.cwl` with the :hblue:`param_` prefix and their initial values. They are internally
translated to a parameter dictionary shared by all tasks in the sub-workflow.
In the execution, :hblue:`%{blah}` in ``opt_args`` is replaced with the actual value in the dictionary.
A loop count is incremented for each iteration and is inserted to the output dataset names, like
:brown:`user.<your_nickname>.blah_<loop_count>_<output>`,
so tasks always produce unique output datasets.
It is possible to specify the loop count explicitly in ``opt_exec`` using :blue:`%{i}`.
In other words, :hblue:`param_i` is reserved and thus cannot be used as user's local variable.

In each iteration, the :blue:`checkpoint` step runs a :brown:`junction` to read outputs from :blue:`inner_work_top`
and :blue:`inner_work_bottom` steps, and produce a json file to update values in the parameter dictionary
and/or make a decision to exit from the loop.
The actual dataset names are passed to the execution string through placeholders, :hblue:`%{DSn}`
in ``opt_exec``, which represents the n-th dataset name.
The json filename must be :brown:`results.json`.
The file contains key-values to update the local variables in the parameter dictionary.
It can also contain a special key-value, :hblue:`to_continue: True`, to repeat the loop.
If :hblue:`to_continue` is :hblue:`False`, the key-value is missing, or :brown:`results.json` is not produced,
the workflow exits from the loop and proceeds to subsequent steps outside of the loop.
It is possible to specify additional output files via :hblue:`---outputs` in ``opt_args`` if the junction step
produces other output files in addition to :brown:`results.json`. Subsequent steps can use those output files
as input.


|br|

Loop + scatter
==================

A loop is sequential iteration of a sub-workflow, while a scatter is a horizontal parallelization of
independent sub-workflows. They can be combined to describe complex workflows.

Running multiple loops in parallel
++++++++++++++++++++++++++++++++++++++

The following example runs multiple loops in parallel.

.. figure:: images/pchain_dag_mloop.png

.. literalinclude:: cwl/multi_loop.cwl
    :language: yaml
    :caption: multi_loop.cwl

The :blue:`work_loop` step has the :hblue:`loop` hint and is scattered over the list of inputs.

Parallel execution of multiple sub-workflows in a single loop
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Here is another example of the loop+scatter combination that has a single loop where
multiple sub-workflows are executed in parallel.

.. figure:: images/pchain_dag_sloop.png

.. literalinclude:: cwl/sequential_loop.cwl
    :language: yaml
    :caption: sequential_loop.cwl

The :blue:`seq_loop` step iterates :brown:`scatter_body.cwl` which defines arrays of parameters
with the the :hblue:`param_` prefix and initial values; :hblue:`param_xxx` and :hblue:`param_yyy`.
The :blue:`parallel_work` step is scattered over the arrays. The arrays are vertically sliced
so that each execution of :brown:`loop_main.cwl` gets only one parameter set.
The :blue:`checkpoint` step takes all outputs from the :blue:`parallel_work` step to update the arrays
and make a decision to continue the loop.

.. literalinclude:: cwl/scatter_body.cwl
    :language: yaml
    :caption: scatter_body.cwl

The sliced values are passed to :brown:`loop_main.cwl` as local variables
:hblue:`param_sliced_x` and :hblue:`param_sliced_y` in the sub-workflow.
In each iteration the :blue:`checkpoint` step above updates the values in the arrays,
so that :hblue:`%{blah}` in ``opt_args`` is replaced with the updated value when the task is actually executed.

.. literalinclude:: cwl/loop_main.cwl
    :language: yaml
    :caption: loop_main.cwl

|br|

Using REANA
======================

The following example offloads a part of workflow to REANA.

.. literalinclude:: cwl/reana.cwl
    :language: yaml
    :caption: reana.cwl

The :blue:`ain` and :blue:`twai` steps are executed on PanDA, while the :blue:`drai` step reads outputs from
those steps and produce the final output on REANA.
The ``run`` filed of a REANA step is :brown:`reana`.
A REANA step is a simplified prun task composed of a single job that
tells input dataset names to the payload through the execution string.
The payload dynamically customizes the sub-workflow description to processes files in those datasets,
submit it to REANA using :doc:`secrets </client/secret>`,
and downloads the results from REANA.

Similarly to junctions, there are only a few parameters in the ``in`` section of a REANA step, as shown below.

.. list-table::
   :header-rows: 1

   * - Parameter
     - Corresponding prun option
   * - opt_inDS
     - Input datasets (a list of strings)
   * - opt_inDsType
     - Types of input datasets (a list of strings. optional)
   * - opt_exec
     - The execution string
   * - opt_containerImage
     - Container image name (string. optional)
   * - opt_args
     - all other prun options except for listed above (string)

The actual dataset names are passed to the execution string through placeholders, :hblue:`%{DSn}`
in ``opt_exec``, which represents the n-th dataset name. Note that the container image in ``opt_containerImage``
submits the sub-workflow description to REANA, so it is generally not the container image that processes input files.
REANA steps are internally executed as prun tasks in PanDA, so that all prun options can be specified in ``opt_args``.


|br|

Using Gitlab CI Pipelines
=====================================

It is possible to integrate Gitlab CI pipelines in workflows.
The following example executes the :blue:`un` step on PanDA, and then feeds the output to the :blue:`deux` step
to trigger a Gitlab CI pipeline.

.. literalinclude:: cwl/gitlab.cwl
    :language: yaml
    :caption: gitlab.cwl

A Gitlab CI step, defined with :brown:`gitlab` in the ``run`` filed, specifies the following parameters
in the ``in`` section.

.. list-table::
   :header-rows: 1

   * - Parameter
     - Description
   * - opt_inDS
     - Input datasets (a list of strings)
   * - opt_input_location
     - The directory name where input files are downloaded (string. optional)
   * - opt_site
     - The site name where the Gitlab CI is reachable (string)
   * - opt_api
     - The API of the Gitlab projects (string, e.g. `https://<hostname>/api/v4/projects`)
   * - opt_projectID
     - The project ID of the gitlab repository where pipelines run (int)
   * - opt_ref
     - The branch name in the repository (string, e.g. master)
   * - opt_triggerToken
     - The key of the trigger token uploaded to :doc:`PanDA secrets </client/secret>` (string)
   * - opt_accessToken
     - The key of the access token uploaded to :doc:`PanDA secrets </client/secret>` (string)

To create trigger and access tokens, see Gitlab documentation pages;
`how to create a trigger token <https://docs.gitlab.com/ee/ci/triggers/>`_
and `how to create an access token <https://docs.gitlab.com/ee/user/project/settings/project_access_tokens.html>`_.
Access tokens require only the **read_api** scope with the **Guest** role.
Note that trigger and access tokens need to be uploaded to :doc:`PanDA secrets </client/secret>`
with keys specified in ``opt_triggerToken`` and ``opt_accessToken, before submitting workflows.


--------------------

|br|



Workflow examples with Snakemake
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Snakemake allows users to describe workflows with Python language syntax and JSON configuration files.

|br|

Common JSON configuration file for following examples
=======================================================

.. literalinclude:: snakemake/examples/config.json
    :language: json
    :caption: config.json

|br|

Simple task chain
======================

.. literalinclude:: snakemake/examples/simple_chain/Snakefile
    :language: yaml
    :caption: simple_chain/Snakefile

``# noinspection`` directives can be used if a workflow is described using PyCharm IDE with SnakeCharm plugin.
In other case these directives can be omitted.
To use parent output as input the user should specify ``rules.${parent_rule_name}.output[0]`` as value
for the corresponding parameter (``opt_inDS``) and specify ``rules.${parent_rule_name}.output`` for ``input`` keyword.

|br|

More complicated chain
========================

.. literalinclude:: snakemake/examples/sig_bg_comb/Snakefile
    :language: yaml
    :caption: sig_bg_comb/Snakefile

To use parameter from another step the user should use the helper function ``param_of`` from
``pandaserver.workflow.snakeparser.utils`` library.
For example, ``param_of("${name_of_parameter}",source=rules.all)``, where ${name_of_parameter} should be replaced by
actual parameter name and source refers to the parameter step.
If step parameter value contains patterns like ``%{SECDS1}`` the helper function ``param_exp`` should be used.
To use multiple inputs (multiple parents) ``input`` keyword should contain all references separated by commas.

|br|

Sub-workflow and parallel execution with scatter
=================================================

.. literalinclude:: snakemake/examples/merge_many/Snakefile
    :language: yaml
    :caption: merge_many/Snakefile

|br|

Using Athena
================

.. literalinclude:: snakemake/examples/athena/Snakefile
    :language: yaml
    :caption: athena/Snakefile

|br|

Conditional workflow
======================

.. literalinclude:: snakemake/examples/cond/Snakefile
    :language: yaml
    :caption: cond/Snakefile

``when`` keyword was added as Snakemake language extension to define step conditions.

|br|

Involving hyperparameter optimization
=======================================

.. literalinclude:: snakemake/examples/hpo/Snakefile
    :language: yaml
    :caption: hpo/Snakefile

|br|

Loops in workflows
====================

.. literalinclude:: snakemake/examples/loop/Snakefile
    :language: yaml
    :caption: loop/Snakefile

.. literalinclude:: snakemake/examples/loop_body/Snakefile
    :language: yaml
    :caption: loop_body/Snakefile

``loop`` keyword was added as Snakemake language extension to define step as a loop.

|br|

Loop + scatter
====================

Running multiple loops in parallel
++++++++++++++++++++++++++++++++++++++

.. literalinclude:: snakemake/examples/multi_loop/Snakefile
    :language: yaml
    :caption: multi_loop/Snakefile

A sub-workflow filename should be specified in ``shell`` keyword.

|br|

Parallel execution of multiple sub-workflows in a single loop
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

.. literalinclude:: snakemake/examples/sequential_loop/Snakefile
    :language: yaml
    :caption: sequential_loop/Snakefile

.. literalinclude:: snakemake/examples/scatter_body/Snakefile
    :language: yaml
    :caption: scatter_body/Snakefile

.. literalinclude:: snakemake/examples/loop_main/Snakefile
    :language: yaml
    :caption: loop_main/Snakefile

Using REANA
=============

.. literalinclude:: snakemake/examples/reana/Snakefile
    :language: yaml
    :caption: reana/Snakefile

-----------------

|br|

Debugging locally
^^^^^^^^^^^^^^^^^^^^^^

Workflow descriptions can be error-prone. It is better to check workflow descriptions before submitting them.
``pchain`` has the ``--check`` option to verify the workflow description locally.
You just need to add the ``--check`` option when running ``pchain``.
For example,

.. prompt:: bash

  pchain --cwl test.cwl --yaml dummy.yaml --outDS user.<your_nickname>.blah --check

which should give a message like

.. code-block:: text

    INFO : uploading workflow sandbox
    INFO : check workflow user.tmaeno.c63e2e54-df9e-402a-8d7b-293b587c4559
    INFO : messages from the server

    internally converted as follows

    ID:0 Name:top Type:prun
      Parent:
      Input:
         opt_args: --outputs seed.txt --nJobs 2 --avoidVP
         opt_exec: echo %RNDM:10 > seed.txt
      Output:
    ...

    INFO : Successfully verified workflow description

Your workflow description is sent to the server to check
whether the options in ``opt_args`` are correct, dependencies among steps are valid,
and input and output data are properly resolved.

-----------------

|br|

Monitoring
^^^^^^^^^^^^^^^^^^^^^^^

-------

|br|

Bookkeeping
^^^^^^^^^^^^^^^^^^^^^^^

:doc:`pbook </client/pbook>` provides the following commands for workflow bookkeeping

.. code-block:: text

    show_workflow
    kill_workflow
    retry_workflow
    finish_workflow
    pause_workflow
    resume_workflow

Please refer to the pbook documentation for their details.

|br|


