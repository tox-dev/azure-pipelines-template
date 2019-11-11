# tox azure-pipeline-template

[![Build Status](https://dev.azure.com/toxdev/azure-pipelines-template/_apis/build/status/tox-dev.azure-pipelines-template?branchName=master)](https://dev.azure.com/toxdev/azure-pipelines-template/_build/latest?definitionId=11&branchName=master)

This a template that will help simplify the [Azure Pipelines](https://azure.microsoft.com/en-gb/services/devops/pipelines/)
configuration when using [tox](https://tox.readthedocs.org) to drive your CI.

# usage

**First configure a github service connection**

It is suggested to use a generic name, such as `github` so forks can also configure the same.

You can find this in `Project Settings` => `Service connections` in the Azure Devops dashboard for your project.
Project settings is located in the bottom left corner of the UI as of 2019-04-30. Below I'm using the endpoint name
`github`.

**To load the template, add this to the beginning of the `azure-pipelines.yml`**

```yaml
resources:
  repositories:
  - repository: tox
    type: github
    endpoint: github
    name: tox-dev/azure-pipelines-template
    ref: refs/tags/0.2
```

this will make the templates in this repository available in the `tox` namespace. Note the ref allows you to pin
the template version you want, you can use ``refs/master`` if you want latest unstable version.

# job templates

## `run-tox-env.yml`

#### Assumptions

tox will run under `Python 3.8`. tox environments generate Junit file under `.tox\junit.{toxenv}.xml`.
Environments tracking coverage data generate will have another tox environment to normalize/merge coverage files.
These should be invoked after test suit runs, and one final time to merge all the sub-coverage files when all
specified tox environments finished (independent their outcome).

### Logic

This job template will run tox for a given set of tox target on given platforms (new in `0.2`).
Features and functionality:

- each specified toxenv target maps to a single Azure Pipelines job, but split over multiple architectures via the
  image matrix (each matrix will set the `image_name` variable to `macOs`, `linux` or `windows`
  depending on the image used)
- make tox available in the job: provision a python (`3.8`) and install a specified tox into that
- provision a python needed for the target tox environment
- provision the target tox environment (create environment, install dependencies)
- invoke the tox target
- if a junit file is found under `.tox\junit.{toxenv}.xml` upload it as test report
- if coverage is requested, run a tox target that should generate the `.tox\coverage.xml` or `.tox\.coverage`
 and upload those as a build artifact (also enqueue a job after all these job succeed to merge the generated
 coverage reports)
- if coverage was requested queue a job that post all toxenv runs will merge all the coverages via a tox target



### example

The following example will run `py36` and `py37` on Windows, Linux and MacOs. It will also invoke
`fix_lint` and `docs` target with `python3.8` on Linux. It will also run the the `coverage` tox environment
for `py37` and `py36`, and then save as build artifacts files `.tox/.coverage` and `.tox/coverage.xml`:

```yaml
jobs:
- template: run-tox-env.yml@tox
  parameters:
    tox_version: ''
    jobs:
      fix_lint: null
      docs: null
      py37:
        image: [linux, windows, macOs]
      py36:
        image: [linux, windows, macOs]
    coverage:
      with_toxenv: 'coverage'
      for_envs: [py37, py36]
```


### parameters

At root level you can control with:

- `tox_version` the tox version specifier to install, this defaults to latest in PyPi (`tox`) - setting it to empty,
- `dependsOn` jobs these set of jobs should depend on
- `before` steps to be run before invoking every tox environment (useful to provision additional dependencies), use
   condition variables for architecture specific content
- `jobs` a map where the key is the tox environment key, while the value is configuration related to that
  environment:

  - the key determines the tox target to run
  - the value contains:

       -  `image` to list an array of targeted architecture, the array elements are mapped as:
          - `linux` - `ubuntu-latest`
          - `windows` - `windows-latest`
          - `osx` - `macOS-latest`
          - otherwise the value if set, fallback to `ubuntu-latest`.

       - `py` - determines the python to provision for running the environment, if not set will be derived from the key:
           - ``py27`` or starts with ``py27-`` - Python 2.7
           - ``py34`` or starts with ``py34-`` - Python 3.4
           - ``py35`` or starts with ``py35-`` - Python 3.5
           - ``py36`` or starts with ``py36-`` - Python 3.6
           - ``py37`` or starts with ``py37-`` - Python 3.7
           - ``py38`` or starts with ``py38-`` - Python 3.8
           - ``pypy`` or starts with ``pypy-`` - PyPy 2
           - ``pypy3`` or starts with ``pypy3-`` - PyPy 3
           - `jython` - Jython is available from under Linux and MacOs.
       - `architecture`: Python architecture (either `x64` or `x86`) with default `x64` (only affects windows)
       - `before` steps to be run before invoking this tox environment (useful to provision additional dependencies)

- `coverage` - if set runs a tox environment (`with_toxenv` - must run with `python3.8`) to normalize coverage data
  (must generate `.tox/.coverage` and `.tox/coverage.xml`) after all environments within ``for_envs``. It also enqueues
   a final job to use `with_toxenv` to merge the coverage files under the name `report_coverage`.

## `publish-pypi.yml`

#### Assumptions
The project is PEP-517 and PEP-518 compatible. A PyPi remote and external feed is configured via Azure Pipelines
project dashboard.

### Logic

This job template will publish the Python package in the current folder (both sdist and wheel) via the PEP-517/8 build
mechanism and twine.

### example

```yaml
- ${{ if startsWith(variables['Build.SourceBranch'], 'refs/tags/') }}:
  - template: publish-pypi.yml@tox
    parameters:
      external_feed: 'toxdev'
      pypi_remote: 'pypi-toxdev'
      dependsOn: [report_coverage, fix_lint, docs]
```

### parameters
- `external_feed` - the external feed to upload
- `pypi_remote` - the pypi remote to upload to
- `dependsOn` - jobs this jobs depends on
