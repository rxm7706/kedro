# Parameters
Project parameters in Kedro are defined inside the `conf` folder in a file that has a filename starting with `parameters`, or are located inside a folder with name starting with `parameters`.
By default, in a new Kedro project, parameters are defined in the `parameters.yml` file, which is located in the project's `conf/base` directory. This file contains a dictionary of key-value pairs, where each key is a parameter name and each value is the corresponding parameter value.
These parameters can serve as input to nodes and are used when running the pipeline. By using parameters, you can make your Kedro pipelines more flexible and easier to configure, since you can change the behaviour of your nodes by modifying the `parameters.yml` file.

## How to use parameters
If you have a group of parameters that determine the hyperparameters of your model, you can define them in a single location such as `conf/base/parameters.yml`. This way, you can keep all your modifications in a centralised location and avoid making changes across multiple parts of your code.

```yaml
step_size: 1
learning_rate: 0.01
```

You can now use the `params:` prefix to reference these parameters in the `node` definition:

```python
def increase_volume(volume, step):
    return volume + step


# in pipeline definition
node(
    func=increase_volume,
    inputs=["input_volume", "params:step_size"],
    outputs="output_volume",
)
```

You can also group your parameters into nested structures and, using the same method above, load them by top-level key:

```yaml
step_size: 1
model_params:
    learning_rate: 0.01
    test_data_ratio: 0.2
    number_of_train_iterations: 10000
```

```python
def train_model(data, model):
    lr = model["learning_rate"]
    test_data_ratio = model["test_data_ratio"]
    iterations = model["number_of_train_iterations"]
    ...


# in pipeline definition
node(
    func=train_model,
    inputs=["input_data", "params:model_params"],
    outputs="output_data",
)
```

Alternatively, you can also pass `parameters` to the node inputs and get access to the entire collection of values inside the node function.

```python
def increase_volume(volume, params):
    step = params["step_size"]
    return volume + step


# in pipeline definition
node(
    func=increase_volume, inputs=["input_volume", "parameters"], outputs="output_volume"
)
```

In both cases, under the hood parameters are added to the Data Catalog through the method `add_feed_dict()` in [`DataCatalog`](/kedro.io.DataCatalog), where they live as `MemoryDataset`s. This method is also what the `KedroContext` class uses when instantiating the catalog.

```{note}
You can use `add_feed_dict()` to inject any other entries into your `DataCatalog` as per your use case.
```

## How to load parameters in code

Parameters project configuration can be loaded by any of the configuration loader classes: `ConfigLoader`, `TemplatedConfigLoader`, and `OmegaConfigLoader`.

The following examples all make use of the default `ConfigLoader` class.

```python
from kedro.config import ConfigLoader
from kedro.framework.project import settings

conf_path = str(project_path / settings.CONF_SOURCE)
conf_loader = ConfigLoader(conf_source=conf_path)
parameters = conf_loader["parameters"]
```

This loads configuration files from any subdirectories in `conf` that have a filename starting with `parameters`, or are located inside a folder with name starting with `parameters`.

Calling `conf_loader[key]` in the example above will throw a `MissingConfigException` error if no configuration files match the given key. But if this is a valid workflow for your application, you can handle it as follows:

```python
from kedro.config import ConfigLoader, MissingConfigException
from kedro.framework.project import settings

conf_path = str(project_path / settings.CONF_SOURCE)
conf_loader = ConfigLoader(conf_source=conf_path)

try:
    parameters = conf_loader["parameters"]
except MissingConfigException:
    parameters = {}
```

```{note}
The `kedro.framework.context.KedroContext` class uses the approach above to load project parameters.
```

[Parameters can then be used on their own or fed in as function inputs](#how-to-use-parameters).

## How to specify parameters at runtime

Kedro also allows you to specify runtime parameters for the `kedro run` CLI command. Use the `--params` command line option and specify a comma-separated list of key-value pairs that will be added to [KedroContext](/kedro.framework.context.KedroContext) parameters and made available to pipeline nodes.

Each key-value pair is split on the first colon or equals sign. The following examples are both valid commands:

```bash
kedro run --params=param_key1:value1,param_key2:2.0  # this will add {"param_key1": "value1", "param_key2": 2} to parameters dictionary
```
```bash
kedro run --params=param_key1=value1,param_key2=2.0
```
Values provided in the CLI take precedence and overwrite parameters specified in configuration files.

* Parameter keys are _always_ treated as strings.
* Parameter values are converted to a float or an integer number if the corresponding conversion succeeds; otherwise, they are also treated as string.

If any extra parameter key and/or value contains spaces, wrap the whole option contents in quotes:

```bash
kedro run --params="key1=value with spaces,key2=value"
```

Since key-value pairs are split on the first colon or equals sign, values can contain colons/equals signs, but keys cannot. These are valid CLI commands:

```bash
kedro run --params=endpoint_url:https://endpoint.example.com
```
```bash
kedro run --params=endpoint_url=https://endpoint.example.com
```
