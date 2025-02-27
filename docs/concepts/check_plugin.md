# check plugin command

The `check_plugin` command would support the following actions:

- `init`: initializes the boilerplate for a custom check plugin using the v2 architecture.
- `release`: generates a release as an _mkp_ package to upload to checkmk site.
- `authenticate` (optional): authenticates with a running site to make sure that the user can upload _mkp_ check plugins. Upon successful authentication, the credentials will be encrypted and stored locally for subsequent commands.
- `install` (optional): installs the specified _mkp_ on a running site.

## Problem

Currently, a user must manually copy their custom check plugin code into a site: `~/local/lib/python3/cmk_addons/plugins/`. They then need to navigate to the website under `Setup > Maitenance > Extension packages`, input meta information, select the files to include, and finally create the package.

This is not very intuitive and interrupts the workflow of developing a check plugin locally. Also, check plugin development at the moment is highly coupled to having access to a running site. A user should be able to develop, test, and package check plugins locally. If they have the proper permissions, then they should also be able to install to a running site.

## API

The following commands should be available:

### init

The initial dialog will pop up to name the plugin:

```console
What do you want to name the check plugin?

Name: my_custom_plugin
```

Then, an interactive dialog would pop up to select what sort of plugin file structure should be generated:

```console
What to include in check plugin?

[x] agent_based (check logic)
[ ] checkman (metadata)
[ ] rulesets (configuration)
[ ] graphing (metric visualization)
...
```

With the following options selected the following structure will be generated:

```
my_custom_plugin/
|_ README.md               # Description about what the check plugin.
|_ agent_based/            # Generated py the command above.
|  |_ my_custom_plugin.py  # Define the check plugin logic here.
|_ dist/                   # Where the packaged plugins are outputed.
```

The `my_custom_plugin.py` file will be prefilled with the following contents:

```python
from typing import TypedDict

from cmk.agent_based.v2 import (
    AgentSection,
    CheckPlugin,
    CheckResult,
    DiscoveryResult,
    Service,
    StringTable,
)


# TODO: determine the content of the example plugin. It's important that
# an external developer gets a "complete" plugin that they can then package
# and run to verify the workflow. Reason: it's much easier to adapt an
# existing plugin then it is to fill in the boilerplate from scratch. This
# would also apply to setting up a special agent in accordance to v2.

def parse_my_custom_plugin(string_table: StringTable):
    # add parsing logic here (remove comment afterwards)
    ...


def discover_my_custom_plugin(section) -> DiscoveryResult:
    # add service discovery logic here (remove comment afterwards)
    yield Service()


class CheckParams(TypedDict):
    # add parameters here (remove comment afterwards)
    ...


def check_my_custom_plugin(params: CheckParams, section) -> CheckResult:
    # add check logic here (remove comment afterwards)
    ...


agent_section_my_custom_plugin = AgentSection(
    name='my_custom_plugin',
    parse_function=parse_my_custom_plugin,
)

check_plugin_my_custom_plugin = CheckPlugin(
    name='my_custom_plugin',
    service_name='my_custom_plugin',
    discovery_function=discover_my_custom_plugin,
    check_function=check_my_custom_plugin,
    check_ruleset_name='my_custom_plugin',
    check_default_parameters=CheckParams(),
)
```

**Note:** by initializing a check plugin with the v2 architecture, an external developer will follow best practices, and will be onboarded more quickly with check plugin development.

### release

Once you have developed and tested the custom check plugin, you can then generate a release as an `mkp` package:

`cmk-ctl check_plugin release my_custom_plugin`

All sub-files in the directory will be selected and the user will have an interactive, prefilled menu:

```console
Enter plugin information

Package name: My Plugin         # derived from the directory name
Check plugin version: 0.1.0     # specify a semantic version (defaults to next bump)
Minimum checkmk version: 2.4.0  # minimum acceptable checkmk version
Author: cmkadmin                # derived from the authenticated user
Description:
Homepage URL:
```

With the following configuration, the check plugin will be packaged and written out to:

```
my_custom_plugin/
|_ ...
|_ dist/
 |_ my_custom_plugin-0.1.0.mkp
```

### authenticate (optional)

`cmk-ctl authenticate`

This command allows the user to authenticate against a running site. The following dialog will be presented to the user:

```console
Authenticate with checkmk site

Site url: https://checkmk.example.com
Username: cmkadmin
Password: ***
```

The TUI authenticates and stores the credentials (encrypted) locally. Credentials will be read from here whenever a admin command is triggered.

### install (optional)

Upload and install the check plugin with:

`cmk-ctl plugin install ./my_custom_plugin`

The first dialog will be to select which site from your authenticated sites to upload to:

```console
Which site(s) to upload to?

[x] https://checkmk.example.com
[ ] https://checkmk-dev.example.com
```

The next dialog will be to specify which version(s):

```console
Which version(s) to upload?

[x] my_custom_plugin-1.0.0.mkp
[ ] my_custom_plugin-0.2.0.mkp
[ ] my_custom_plugin-0.1.0.mkp
```

The command will try and upload the check plugin via the REST API. If the user has adequate permissions, the request handler will unpack the check plugin and store the source code under the appropriate site path.
