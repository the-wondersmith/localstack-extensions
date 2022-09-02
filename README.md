# LocalStack Extensions (beta)

<p align="center">
  <img src="https://user-images.githubusercontent.com/3996682/184503940-c30bfcac-e049-4ee4-b905-207b340111d1.png" alt="LocalStack Extensions">
</p>

With LocalStack 1.0 we have introduced LocalStack Extensions that allow
developers to extend and customize LocalStack. Both the feature and the API
are currently experimental and may be subject to change.

## Using Extensions

Extensions are a LocalStack Pro feature.
To use and install extensions, use the CLI to first log in to your account

```console
$ localstack login
Please provide your login credentials below
Username: ...
```

```console
$ localstack extensions --help

Usage: localstack extensions [OPTIONS] COMMAND [ARGS]...

  Manage LocalStack extensions (beta)

Options:
  --help  Show this message and exit.

Commands:
  init       Initialize the LocalStack extensions environment
  install    Install a LocalStack extension
  uninstall  Remove a LocalStack extension
```

To install an extension, specify the name of the pip dependency that contains
the extension. For example, for the official Stripe extension, you can either
use the package distributed on pypi:

```console
$ localstack extensions install localstack-extensions-stripe
```

or you can install it directly from this git repository

```console
$ localstack extensions install "git+https://github.com/localstack/localstack-extensions/#egg=localstack-extensions-stripe&subdirectory=stripe"
```

## Developing Extensions

### The extensions API

LocalStack exposes a Python API for building extensions that can be found in
the core codebase in
[`localstack.extensions.api`](https://github.com/localstack/localstack/tree/master/localstack/extensions/api).

The basic interface to implement is as follows:

```python
class Extension(BaseExtension):
    """
    An extension that is loaded into LocalStack dynamically. The method
    execution order of an extension is as follows:

    - on_extension_load
    - on_platform_start
    - update_gateway_routes
    - update_request_handlers
    - update_response_handlers
    - on_platform_ready
    """

    namespace: str = "localstack.extensions"
    """The namespace of all basic localstack extensions."""

    name: str
    """The unique name of the extension set by the implementing class."""

    def on_extension_load(self):
        """
        Called when LocalStack loads the extension.
        """
        pass

    def on_platform_start(self):
        """
        Called when LocalStack starts the main runtime.
        """
        pass

    def update_gateway_routes(self, router: Router[RouteHandler]):
        """
        Called with the Router attached to the LocalStack gateway. Overwrite this to add or update routes.
        :param router: the Router attached in the gateway
        """
        pass

    def update_request_handlers(self, handlers: CompositeHandler):
        """
        Called with the custom request handlers of the LocalStack gateway. Overwrite this to add or update handlers.
        :param handlers: custom request handlers of the gateway
        """
        pass

    def update_response_handlers(self, handlers: CompositeResponseHandler):
        """
        Called with the custom response handlers of the LocalStack gateway. Overwrite this to add or update handlers.
        :param handlers: custom response handlers of the gateway
        """
        pass

    def on_platform_ready(self):
        """
        Called when LocalStack is ready and the Ready marker has been printed.
        """
        pass
```

A minimal example would look like this:

```python
import logging
from localstack.extensions.api import Extension

LOG = logging.getLogger(__name__)

class ReadyAnnoucerExtension(Extension):
    name = "my_ready_annoucer"

    def on_platform_ready(self):
    	LOG.info("my plugin is loaded and localstack is ready to roll!")
```

### Package your Extension

Your extensions needs to be packaged as a Python distribution with a
`setup.cfg` or `setup.py` config. LocalStack uses the
[Plux](https://github.com/localstack/plux) code loading framework to load your
code from a Python [entry point](https://packaging.python.org/en/latest/specifications/entry-points/).
You can either use Plux to discover the entrypoints from your code when
building and publishing your distribution, or manually define them as in the
example below.

A minimal `setup.cfg` for the extension above could look like this:

```ini
[metadata]
name = localstack-extension-ready-announcer
description = LocalStack extension that logs when LocalStack is ready to receive requests
author = Your Name
author_email = your@email.com
url = https://link-to-your-project

[options]
zip_safe = False
packages = find:
install_requires =
    localstack>=1.0.0

[options.entry_points]
localstack.extensions =
    my_ready_annoucer = localstack_annoucer.extension:ReadyAnnoucerExtension
```

The entry point group is the Plux namespace `locastack.extensions`, and the
entry point name is the plugin name `my_ready_announcer`. The object
reference points to the plugin class.


### Using the extensions CLI

The extensions CLI has a set of developer commands that allow you to create new extensions, and toggle local dev mode for extensions.
Extensions that are toggled for developer mode will be mounted into the localstack container so you don't need to re-install them every time you change something.

```console
Usage: localstack extensions dev [OPTIONS] COMMAND [ARGS]...

  Developer tools for developing Localstack extensions

Options:
  --help  Show this message and exit.

Commands:
  disable  Disables an extension on the host for developer mode.
  enable   Enables an extension on the host for developer mode.
  list     List LocalStack extensions for which dev mode is enabled.
  new      Create a new LocalStack extension from the official extension...
```

#### Creating a new extensions

First, create a new extensions from a template:

```console
 % localstack extensions dev new
project_name [My LocalStack Extension]: 
project_short_description [All the boilerplate you need to create a LocalStack extension.]: 
project_slug [my-localstack-extension]: 
module_name [my_localstack_extension]: 
full_name [Jane Doe]: 
email [jane@example.com]: 
github_username [janedoe]: 
version [0.1.0]: 
```

This will create a new python project with the following layout:

```
my-localstack-extension
├── Makefile
├── my_localstack_extension
│   ├── extension.py
│   └── __init__.py
├── README.md
├── setup.cfg
└── setup.py
```

Then run `make install` in the newly created project to make a distribution package.

#### Start LocalStack with the extension

To start LocalStack with the extension in dev mode, first enable it by running:

```console
localstack extensions dev enable ./my-localstack-extension
```

Then, start LocalStack with `EXTENSIONS_DEV_MODE=1`

```console
EXTENSION_DEV_MODE=1 LOCALSTACK_API_KEY=... localstack start
```

In the LocalStack logs you should then see something like:
```
==================================================
👷 LocalStack extension developer mode enabled 🏗
- mounting extension /opt/code/extensions/my-localstack-extension
Resuming normal execution, ...
==================================================
```

Now, when you make changes to your extensions, you just need to restart LocalStack and the changes will be picked up by LocalStack automatically.
