# fnExchange Sample Plugin
This repository serves as a sample project for developing (and publishing)
fnExchange plugins. In order to create a new project, you can simply clone
this project and follow the instructions below to get started instantly.

## About fnExchange
fnExchange is a scalable, open source API layer (also called an API "router")
that provides a consistent proxy web interface for invoking various web APIs
without the caller having to write separate, special-purpose code for each of
them within the application.

More details about fnExchange can be found at the fnExchange GitHub
[repo](https://github.com/dnif/fnExchange).

## Sample Repo Structure

Other than code, this project consists of the following files that you should be
updating:

- README.md: Contains the documentation / instructions for your plugin
- LICENSE: Describes the license under which the plugin is made available
    (if applicable)
- requirements.txt: The file listing package requirements for your plugin
- setup.py: File for helping publish the plugin via pip. Instructions for
    editing the file have been added to the file in the form of comments.
    **Note**: You probably want to use ths file even if you're not going
    to open source your plugin or not going to make it available via the
    PyPi index (`pip install <packagename>`), because you could then still
    do a local install using `pip install .`
- setup.cfg: describes configuration used by setup.py commands. Read the
    comments in the file for more details.

Lastly, the python module greetings.py contains the actual sample `GreetingsPlugin`.
This is where your shiny new python code (packages / modules) will go.
Please make sure to namespace your plugin well in case you're using common
module names.

## Sample Plugin: Greetings!
GreetingsPlugin provides an interface to generate greetings in different languages 
for given users (provided their names and locales). At this time, only the
following locales are supported: "en-us", "hi-in".

### Configuration
This plugin requires a `greeter` name to be configured when initializing the plugin.

### Usage
This plugin can be used by adding the following configuration to the `fnexchange.yml`
configuration file under `plugins_enabled`:

```yaml
...
  plugins_enabled:
    ...
    greetings:
      class_name: 'greetings.GreetingsPlugin'
      config:
        greeter: 'John Doe'
    ...
...
```

# fnExchange Plugin Development Guide

## Overview
fnExchange plugins are the components that actually provide implementations
to perform arbitrary actions based on data. The fnExchange server "routes"
requests (actually, just the request payload) to the specified plugin, and
the plugin handles the processing, and returns a response payload.

## Components
While developing and using a plugin, there are three critical pieces to keep
in mind:
1. The request payload structure (interface)
2. The response payload structure (interface)
3. The actual implementation (code)

It is very important to pay attention to the request and response payload
interfaces. fnExchange itself supports _any_ valid JSON in the payload and
imposes no restrictions whatsoever, but if the caller has to remember to send
wildly different data structures to different plugins, it will become unwieldy
to manage over time. It is therefore recommended to follow a (relatively)
standard structure for your application.

## Payload Structure: fnExchange (DNIF) defaults
fnExchange has been originally created to allow more powerful control / actions
for dnif. For our purposes, we use the following payload formats for requests
and responses.

If you are developing fnExchange plugins compatible with DNIF (more likely than not),
you are required to follow the above request and response payload formats.

### Request Payload
The request payload consists of a dict with two keys:
- `elements`:
    This is a list of dicts, each dict representing one atomic data 'element'
    to compute upon. The elements key is a list to enable actions over multiple
    data points in a single call instead of having to make several calls to
    fnExchange.
    
    One way to look at elements is to consider it equivalent to a "table", where
    rows are represented by each individual `element` and the keys of the element
    are columns. 

- `metadata`:
    This is a dict with arbitrary key-value pairs. This key is designed to provide
    any request-level configuration that the plugin might need to function correctly.
    While this data could also be passed in within each element of `elements`,
    it may be more fitting to provide common data here. As an example, if the
    GreetingsPlugin was to return a greeting with an advertisement like so:
    "Hello, Emma! My name is John. Today's special dish is Burrito", it would
    make sense to pass in Burrito just once in metadata instead of in each element
    row.

```json
{
  "metadata": {
    "key1": "value1",
    "key2": 2.3
  },
  "elements": [
    {
      "keyX": "valueX",
      "keyY": 1.23
    }
  ]
}
```

### Response Payload
The response payload also follows the same structure. It consists of a dict with
two keys:

- `elements`:
    This is a list of dicts, each dict representing one atomic data 'element' response.
    **The number of elements in the response is independent of the number of elements
    in the request.**

- `metadata`:
    This is a dict with arbitrary key-value pairs. This key is designed to provide
    any response-level information that the plugin emits. This could be used to
    transmit plugin state information, or metadata about the current request (like
    success/failure/errors, etc.)

```json
{
  "metadata": {
    "key1": "value1",
    "key2": 2.3
  },
  "elements": [
    {
      "keyX": "valueX",
      "keyY": 1.23
    }
  ]
}
```

If you're using fnExchange for your own application (and developing plugins for your
custom use), you can still use the same format unless you require something specifically
tailored for your needs. Using a standard format helps interop and allows you to directly
use any plugins developed for DNIF out of the box. 


## Show me the code!
Alright, let's get to the actual plugin development. fnExchange plugins are simply normal
Python classes. These may, for convenience extend AbstractPlugin (in order to not have to
write the `__init__` method), but that is not mandatory.

```python
from fnexchange.core.plugins import AbstractPlugin
class MyPlugin1(AbstractPlugin):
    def say_hello(self, payload):
        return {
            "elements": [
                {"name": "Emma", "greeting": "Hello Emma!"},
            ],
            "metadata": {}
        }

# same as
class MyPlugin2(object):
    def __init__(self, plugin_config):
        self.config = plugin_config

    def say_hello(self, payload):
        return {
            "elements": [
                {"name": "Emma", "greeting": "Hello Emma!"},
            ],
            "metadata": {}
        }
```

Any action that the plugin provides is just a standard public method on the Plugin
class. So in the above example, "say_hello" is an action.


## Using Configuration variables
Your plugins will more often than not require some configuration value to function at
all. Examples of these are: oAuth (or other security) tokens when making web service
calls, or perhaps something like a numerical constant that is required for some
computation.

These configuration variables are to be added in the `fnexchange.yml` (as described above
in Usage), and these are available to the Plugin instance as `self.config.CONFIGNAME`

Here's a simple example of a plugin that accepts a request with a list of log statements
and filters the ones we desire (grep)

```python
from fnexchange.core.plugins import AbstractPlugin
class Grepper(AbstractPlugin):
    def grep(self, payload):
        return {
            "metadata": {},
            "elements": filter(lambda x: self.config.search_word in x["log"], payload["elements"]),
        }
```

See how easy these plugins are ? :)
