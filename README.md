# tox azure-pipeline-template

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
    ref: refs/tags/0.1
```

this will make the templates in this repository available in the `tox` namespace.

# job templates

## `run-tox-env.yml`
This job template will run tox for a given set of tox targets on given platform (new in `0.1`). 
Features and functionality:

- each specified toxenv target maps to a single Azure Pipelines job
- provision a python for the job
- install tox into that python
- provision the target tox environment (create environment, install dependencies)
- invoke the tox target
- if a junit file is found under `.tox\junit.{toxenv}.xml` upload it as test report
- if a coverage file is found under `.tox\coverage.xml` or `.tox\.coverage` upload it as build artifact

### example

The following example will run `py36` and `py37` on Windows, Linux and MacOs. It will also invoke
`fix_lint` and `docs` target with `python3.7` on Linux. Note how the root level name can be used
to automatically specify the target platform (defaults to Linux).
               
```yaml
jobs:
- template: run-tox-env.yml@tox
  parameters:
    jobs:
      windows:
        toxenvs:
        - py37
        - py36
      linux:
        toxenvs:
        - py37
        - py36
      macOs:
        toxenvs:
        - py37
        - py27
      check:
        py: '3.7'
        toxenvs:
        - fix_lint
        - docs
```


### parameters

At root level has a single ``jobs`` key. Inside this you can enlist groups of targets as maps. 
The key is the name of the group. The values configure the target:
 
- `toxenvs`: the list of `tox` environment names to run; must either:
  - be equal to: `py27`, `py34`, `py35`, `py36`, `py37`, `py38`
  - start with: `py27-`, `py34-`, `py35-`, `py36-`, `py37-`, `py38-`

- `image`: specify the Azure pipelines image to use (determines the OS target); if not specified
  we'll use the groups key name to assign one:
  - `linux` - `Ubuntu-16.04`
  - `windows` - `windows-2019`
  - `osx` - `macOS-latest`
  - otherwise `Ubuntu-16.04`
- `architecture`: either `x64` or `x86`) with default `x64` (only affects windows)
- `coverage`: if set after running the test suite tox will run this tox target to generate a coverage report.

Note: for now, `python3.8` is only available on linux -- it is installed from
[deadsnakes](https://github.com/deadsnakes).

## `merge-coverage.yml`

This job template will download coverage files attached to the build (uploaded by `run-tox-env.yml`)
and use target tox environment configured to generate a unified report. This then will be uploaded
to the Azure Pipelines coverage report. 

### example

```yaml
- template: merge-coverage.yml@tox
  parameters:
    coverage: 'coverage'
    dependsOn:
    - windows
    - linux
    - macOs
```

### parameters
- `coverage` - tox target that generates the unified coverage report (default `coverage`)
- `dependsOn` - environments this job depends on 

## `publish-pypi.yml`

This job template will publish the Python package in the current folder (both sdist and wheel)
via the PEP-517/8 build mechanism and twine. 

### example

```yaml
- ${{ if startsWith(variables['Build.SourceBranch'], 'refs/tags/') }}:
  - template: publish-pypi.yml@tox
    parameters:
    - external_feed: 'toxdev'
    - pypi_remote: 'pypi-toxdev'
    - dependsOn:
      - check
      - report_coverage
```

### parameters
- `external_feed` - the external feed to upload
- `pypi_remote` - the pypi remote to upload to 
- `dependsOn` - environments this job depends on 
