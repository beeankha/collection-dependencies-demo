### Sniff requirements from collections

This tool is to build out data for what requirements different collections
have. It is ultimately intended to go into `ansible-builder` but only
to be ran once / initially as a means of bootstrapping all the metadata
for the collections out in the wild.

https://github.com/ansible/ansible-builder

### How to use

You need a folder which has collections in it. This could be `~/.ansible/collections`.
Other tools here are good for producing such folders with different reference
collection sets. See `cloner` for AWX deps & ACD, and `everything` if you're looking
to have a very bad time.

Take that folder and throw it into the script.

```
brew install libvirt  # only macos
pip install boto3 botocore linode_api4 libvirt-python pymongo pynetbox msrest msrestazure pytz ucsmsdk python-memcached redis
python sniff_req.py target
```

The pip installs are because some modules will error on import without libraries.
Not all plugins will be sniffed, due to unidentified bugs.
This script takes about 1 minute to do.

### Preliminary Findings

#### Declared Requirements

For the raw output, see the file in this folder `discovered.json`.

#### Import Errors

Evaluating the documentation is done via importing the plugins (or modules).
Because of that, an incidental consequence is that we identify the plugins
and modules which cannot be ordinarily imported.

Since there is not a huge number of them, they are pasted here.

```
{
  "arista.eos": [
    "collection metadata was not loaded for collection arista.eos"
  ],
  "netbox.netbox": [
    "No module named 'pynetbox'",
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "amazon.aws": [
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "azure.azcollection": [
    "No module named 'ansible.module_utils.azure_rm_common'",
    "No module named 'ansible.module_utils.azure_rm_common_ext'",
    "bad magic number in 'ansible.module_utils.network': b'\\x03\\xf3\\r\\n'"
  ],
  "dellemc_networking.os10": [
    "No module named 'ansible_collections.dellemc'"
  ],
  "google.cloud": [
    "bad magic number in 'ansible.module_utils.gcp_utils': b'\\x03\\xf3\\r\\n'"
  ],
  "cisco.iosxr": [
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "cisco.nxos": [
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "cisco.ios": [
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "cisco.asa": [
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "cisco.ucs": [
    "No module named 'ucsmsdk'"
  ],
  "ansible.netcommon": [
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "vyos.vyos": [
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "junipernetworks.junos": [
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "community.general": [
    "No module named 'ansible_collections.community.general.plugins.connection.kubectl'"
  ],
  "community.network": [
    "collection metadata was not loaded for collection ansible.netcommon"
  ],
  "fortinet.fortios": [
    "bad magic number in 'ansible.module_utils.network': b'\\x03\\xf3\\r\\n'"
  ]
}
```

##### Local Dependency

Some of these have resolutions, `pynetbox` just needed to be added to the
pip installs.

##### Azure module_utils from base

The `azure.azcollection` seems genuinely concerning.
This is not due to being out-of-date, because the entire point of creating
the `target` folder was the clone with the most recent active branch.

For sanity, all these issues can be replicated manually via:

```
$ PYTHONPATH=target:$PYTHONPATH python -c 'import ansible_collections.azure.azcollection.plugins.modules.azure_rm_mysqlserver_info'
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/Users/alancoding/Documents/repos/collection-dependencies-demo/target/ansible_collections/azure/azcollection/plugins/modules/azure_rm_mysqlserver_info.py", line 156, in <module>
    from ..module_utils.azure_rm_common import AzureRMModuleBase
ModuleNotFoundError: No module named 'ansible.module_utils.azure_rm_common'
```

This also isn't obviously necessary, since the collection itself hosts the
module_utils such as `azure_rm_common`.

Changing that example file import to:

```
from ..module_utils.azure_rm_common import AzureRMModuleBase
```

makes that import command work. Put up PR at

https://github.com/ansible-collections/azure/pull/173

##### Bad magic number from module_utils imported from base

The "bad magic number in" errors look to be the same category of error,
where they import module_utils that should be inside the collection from
the old location in Ansible 2.9. Such as:

> bad magic number in 'ansible.module_utils.network'

That module shouldn't exist, and should be moved to the netcommon collection.

This error was converted to other errors by running the
`find target -name '*.pyc' -delete` kind of command in the Ansible core
directory.

##### Metadata not loaded

The error of "collection metadata was not loaded for collection ansible.netcommon"
reflects something much more like a code bug.

https://github.com/ansible/ansible/blob/51f6d129cbb30f42c445f7e2fecba68fe02d6f85/lib/ansible/utils/collection_loader/_collection_finder.py#L968

This is only a secondary error from an original obfuscated error of

```
Traceback (most recent call last):
  File "<frozen importlib._bootstrap>", line 900, in _find_spec
AttributeError: '_AnsibleCollectionFinder' object has no attribute 'find_spec'
```

Ultimately, the cause was found to be due to a prior import. Replicate
locally by running:

```
$ PYTHONPATH=target:$PYTHONPATH python -c 'import ansible_collections.arista.eos.plugins.action.eos, ansible_collections.arista.eos.plugins.modules.eos_lldp_interfaces'
Traceback (most recent call last):
  File "<frozen importlib._bootstrap>", line 900, in _find_spec
AttributeError: '_AnsibleCollectionFinder' object has no attribute 'find_spec'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "<frozen importlib._bootstrap>", line 983, in _find_and_load
  File "<frozen importlib._bootstrap>", line 963, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 902, in _find_spec
  File "<frozen importlib._bootstrap>", line 876, in _find_spec_legacy
  File "/Users/alancoding/Documents/repos/ansible/lib/ansible/utils/collection_loader/_collection_finder.py", line 180, in find_module
    return _AnsibleCollectionLoader(fullname=fullname, path_list=path)
  File "/Users/alancoding/Documents/repos/ansible/lib/ansible/utils/collection_loader/_collection_finder.py", line 272, in __init__
    self._subpackage_search_paths = self._get_subpackage_search_paths(self._candidate_paths)
  File "/Users/alancoding/Documents/repos/ansible/lib/ansible/utils/collection_loader/_collection_finder.py", line 565, in _get_subpackage_search_paths
    collection_meta = _get_collection_metadata(collection_name)
  File "/Users/alancoding/Documents/repos/ansible/lib/ansible/utils/collection_loader/_collection_finder.py", line 968, in _get_collection_metadata
    raise ValueError('collection metadata was not loaded for collection {0}'.format(collection_name))
ValueError: collection metadata was not loaded for collection arista.eos
```

The source of the monkeypatch was then further hunted to:

 - ansible_collections.ansible.netcommon.plugins.action.network
 - ansible_collections.ansible.netcommon.plugins.action.xxxxx

It looks like a very deep issue with how the ActionBase module does things,
because even if all the logic from the network-specific action modules are
removed, the behavior persists. Oddly, action modules from other collections
will not cause this sort of trouble.

Solution is to pass over these action modules for now.

##### Kubectl move bug

The failed kubectl import also seems to be a bug:

https://github.com/ansible-collections/community.general/pull/5#issuecomment-648561220

##### Parsing failures

Some DOCUMENTATION could not be parsed, in some cases converting to a
raw string fixed it.

https://github.com/ansible-collections/google.cloud/issues/248

Single-line replication:

```
python -c "import pathlib, yaml; yaml.safe_load(pathlib.Path('sniff_req/gcp_test.yml').read_text())"
```
