# ServiceX_frontend

 Client access library for ServiceX

[![GitHub Actions Status](https://github.com/ssl-hep/ServiceX_frontend/workflows/CI/CD/badge.svg?branch=master)](https://github.com/ssl-hep/ServiceX_frontend/actions)
[![Code Coverage](https://codecov.io/gh/ssl-hep/ServiceX_frontend/graph/badge.svg)](https://codecov.io/gh/ssl-hep/ServiceX_frontend)

[![PyPI version](https://badge.fury.io/py/servicex.svg)](https://badge.fury.io/py/servicex)
[![Supported Python versions](https://img.shields.io/pypi/pyversions/servicex.svg)](https://pypi.org/project/servicex/)

## Introduction

Given you have a selection string, this library will manage submitting it to a ServiceX instance and retrieving the data locally for you.
The selection string is often generated by another front-end library, for example:

- [func_adl.xAOD](https://github.com/iris-hep/func_adl_xAOD) (for ATLAS xAOD's)
- [func_adl.uproot](https://github.com/iris-hep/func_adl.uproot) (for flat ntuples)
- [tcut_to_castle](https://pypi.org/project/tcut-to-qastle/) (translates `TCut` like syntax into a `servicex` query - should work for both)

## Prerequisites

Before you can use this library you'll need:

- An environment based on python 3.6 or later
- A `ServiceX` end-point. For example, `http://localhost:5000/servicex`, if `ServiceX` is running on a local `k8` cluster and the proper ports are open, or the public servicex instance (contact IRIS-HEP at xxx if you are part of the LHC to request an account, or with help setting up an instance).

### How to access your endpoint

The `servicex` library searches for configuration information in several locations to determine what end-point it should connect to, in the following order:

1. A `.servicex` file in the current working directory
1. A `.servicex` file in the user's home directory (`$HOME` on Linux and Mac, and your profile
   directory on Windows).
1. The `config_defaults.yaml` file distributed with the `servicex` package.

If no endpoint is specified, then the library defaults to the developer endpoint, which is `http://localhost:5000` for the web-service API, and `localhost:9000` for the `minio` endpoint. No passwords are required.

Create a `.servicex` file, in the `yaml` format, in the appropriate place for your work that contains the following:

```yaml
api_endpoints:
  - endpoint: <your-endpoint>
    email: <api-email>
    password: <api-password>
    type: xaod
```

All strings are expanded using python's [os.path.expand](https://docs.python.org/3/library/os.path.html#os.path.expandvars) method - so `$NAME` and `${NAME}` will work to expand existing environment variables.

You can list multiple end points by repeating the block of 4 dictionary items, but using a different type. For example, `uproot`.

Finally, you can create the objects `ServiceXAdaptor` and `MinioAdaptor` by hand in your code, passing them as arguments to `ServiceXDataset` and inject custom endpoints and credentials, avoiding the configuration system. This is probably only useful for advanced users.

## Usage

The following lines will return a `pandas.DataFrame` containing all the jet pT's from an ATLAS xAOD file containing Z->ee Monte Carlo:

```python
    from servicex import ServiceX
    query = "(call ResultTTree (call Select (call SelectMany (call EventDataset (list 'localds:bogus')) (lambda (list e) (call (attr e 'Jets') 'AntiKt4EMTopoJets'))) (lambda (list j) (/ (call (attr j 'pt')) 1000.0))) (list 'JetPt') 'analysis' 'junk.root')"
    dataset = "mc15_13TeV:mc15_13TeV.361106.PowhegPythia8EvtGen_AZNLOCTEQ6L1_Zee.merge.DAOD_STDM3.e3601_s2576_s2132_r6630_r6264_p2363_tid05630052_00"
    ds = ServiceXDataset(dataset, 'xaod')
    r = ds.get_data_pandas_df(query)
    print(r)
```

And the output in a terminal window from running the above script (takes about 1-2 minutes to complete):

```bash
python scripts/run_test.py http://localhost:5000/servicex
            JetPt
entry
0       38.065707
1       31.967096
2        7.881337
3        6.669581
4        5.624053
...           ...
710183  42.926141
710184  30.815709
710185   6.348002
710186   5.472711
710187   5.212714

[11355980 rows x 1 columns]
```

If your query is badly formed or there is an other problem with the backend, an exception will be thrown with information about the error.

If you'd like to be able to submit multiple queries and have them run on the `ServiceX` back end in parallel, it is best to use the `asyncio` interface, which has the identical signature, but is called `get_data_pandas_df_async`.

For documentation of `get_data` and `get_data_async` see the `servicex.py` source file.

## Configuration

As mentioned above, the `.servicex` file is read to pull a configuration. The search path for this file:

1. Your current working directory
2. Your home directory

The file can contain an `api_endpoint` as mentioned above. In addition the other following things can be put in:

- `cache_path`: Location where queries, data, and a record of queries are written. This should be an absolute path the person running the library has r/w access to. On windows, make sure to escape `\` - and best to follow standard `yaml` conventions and put the path in quotes - especially if it contains a space. Top level yaml item (don't indent it accidentally!). Defaults to `/tmp/servicex` (with the temp directory as appropriate for your platform) Examples:
  - Windows: `cache_path: "C:\\Users\\gordo\\Desktop\\cacheme"`
  - Linux: `cache_path: "/home/servicex-cache"`

- `minio_endpoint`, `minio_username`, `minio_password` - these are only interesting if you are using a pre-RC2 release of `servicex` - when the `minio` information wasn't part of the API exchange. This feature is depreciated and will be removed around the time `servicex` moves to RC3.

All strings are expanded using python's [os.path.expand](https://docs.python.org/3/library/os.path.html#os.path.expandvars) method - so `$NAME` and `${NAME}` will work to expand existing environment variables.

## Features

Implemented:

- Accepts a `qastle` formatted query
- Exceptions are used to report back errors of all sorts from the service to the user's code.
- Data is return in the following forms:
  - `pandas.DataFrame` an in process DataFrame of all the data requested
  - `awkward` an in process `JaggedArray` or dictionary of `JaggedArray`s
  - A list of root files that can be opened with `uproot` and used as desired.
  - Not all output formats are compatible with all transformations.
- Complete returned data must fit in the process' memory
- Run in an async or a non-async environment and non-async methods will accommodate automatically (including `jupyter` notebooks).
- Support up to 100 simultaneous queries from a laptop-like front end without overwhelming the local machine (hopefully ServiceX will be overwhelmed!)
- Start downloading files as soon as they are ready (before ServiceX is done with the complete transform).
- It has been tested to run against 100 datasets with multiple simultaneous queries.
- It supports local caching of query data
- It will provide feedback on progress.
- Configuration files supported so that user identification information does not have to be checked
  into repositories.

## Testing

This code has been tested in several environments:

- Windows, Linux, MacOS
- Python 3.6, 3.7, 3.8
- Jupyter Notebooks (not automated), regular python command-line invoked source files

## API

Everything is based around the `ServiceXDataset` object. Below is the documentation for the most common parameters.

```python
  ServiceXDataset(dataset: str,
                 backend_type: Optional[str] = None,
                 image: str = 'sslhep/servicex_func_adl_xaod_transformer:v0.4',
                 max_workers: int = 20,
                 servicex_adaptor: ServiceXAdaptor = None,
                 minio_adaptor: MinioAdaptor = None,
                 cache_adaptor: Optional[Cache] = None,
                 status_callback_factory: Optional[StatusUpdateFactory] = _run_default_wrapper,
                 local_log: log_adaptor = None,
                 session_generator: Callable[[], Awaitable[aiohttp.ClientSession]] = None,
                 config_adaptor: ConfigView = None):
  '''
      Create and configure a ServiceX object for a dataset.

      Arguments

          dataset                     Name of a dataset from which queries will be selected.
          backend_type                The type of backend. Used only if we need to find an
                                      end-point. If we do not have a `servicex_adaptor` then this
                                      cannot be null. Possible types are `uproot`, `xaod`,
                                      and anything that finds a match in the `.servicex` file.
          image                       Name of transformer image to use to transform the data
          max_workers                 Maximum number of transformers to run simultaneously on
                                      ServiceX.
          servicex_adaptor            Object to control communication with the servicex instance
                                      at a particular ip address with certain login credentials.
                                      Will be configured via the `.servicex` file by default.
          minio_adaptor               Object to control communication with the minio servicex
                                      instance. By default configured with values from the
                                      `.servicex` file.
          cache_adaptor               Runs the caching for data and queries that are sent up and
                                      down.
          status_callback_factory     Factory to create a status notification callback for each
                                      query. One is created per query.
          local_log                   Log adaptor for logging.
          session_generator           If you want to control the `ClientSession` object that
                                      is used for callbacks. Otherwise a single one for all
                                      `servicex` queries is used.
          config_adaptor              Control how configuration options are read from the
                                      `.servicex` file.

      Notes:

          -  The `status_callback` argument, by default, uses the `tqdm` library to render
             progress bars in a terminal window or a graphic in a Jupyter notebook (with proper
             jupyter extensions installed). If `status_callback` is specified as None, no
             updates will be rendered. A custom callback function can also be specified which
             takes `(total_files, transformed, downloaded, skipped)` as an argument. The
             `total_files` parameter may be `None` until the system knows how many files need to
             be processed (and some files can even be completed before that is known).
  '''
 ```

To get the data use one of the `get_data` method. They all have the same API, differing only by what they return.

```python
 |  get_data_awkward_async(self, selection_query: str) -> Dict[bytes, Union[awkward.array.jagged.JaggedArray, numpy.ndarray]]
 |      Fetch query data from ServiceX matching `selection_query` and return it as
 |      dictionary of awkward arrays, an entry for each column. The data is uniquely
 |      ordered (the same query will always return the same order).
 |
 |  get_data_awkward(self, selection_query: str) -> Dict[bytes, Union[awkward.array.jagged.JaggedArray, numpy.ndarray]]
 |      Fetch query data from ServiceX matching `selection_query` and return it as
 |      dictionary of awkward arrays, an entry for each column. The data is uniquely
 |      ordered (the same query will always return the same order).
```

Each data type comes in a pair - an `async` version and a synchronous version.

- `get_data_awkward_async, get_data_awkward` - Returns a dictionary of the requested data as `numpy` or `JaggedArray` objects.
- `get_data_rootfiles`, `get_data_rootfiles_async` - Returns a list of locally download files (as `pathlib.Path` objects) containing the requested data. Suitable for opening with [`ROOT::TFile`](https://root.cern.ch/doc/master/classTFile.html) or [`uproot`](https://github.com/scikit-hep/uproot).
- `get_data_pandas_df`, `get_data_pandas_df_async` - Returns the data as a `pandas` `DataFrame`. This will fail if the data you've requested has any structure (e.g. is hierarchical, like a single entry for each event, and each event may have some number of jets).
- `get_data_parquet`, `get_data_parquet_async` - Returns a list of files locally downloaded that can be read by any parquet tools.

## Development

For any changes please feel free to submit pull requests! We are using the `gitlab` workflow: the `master` branch represents the latests updates that pass all tests working towards the next version of the software. Any PR's should be based off the most recent version of `master` if they are for new features. Each release is frozen on a dedicated release branch, e.g. v2.0.0. If a bug fix needs to be applied to an existing release, submit a PR to master mentioning the affected version(s). After the PR is merged to master, it will be applied to the relevant release branch(es) using git cherry-pick.

To do development please setup your environment with the following steps:

1. A python 3.7 development environment
2. Fork/Pull down this package, XX
3. `python -m pip install -e .[test]`
4. Run the tests to make sure everything is good: `pytest`.

Then add tests as you develop. When you are done, submit a pull request with any required changes to the documentation and the online tests will run.

### To create a release branch

```bash
get checkout 2.0.0
get switch -c v2.0.0
git push
```
