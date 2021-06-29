# JupyterHub Idle Culler Service

[![GitHub Workflow Status - Test](https://img.shields.io/github/workflow/status/jupyterhub/jupyterhub-idle-culler/Test?logo=github&label=tests)](https://github.com/jupyterhub/jupyterhub-idle-culler/actions)
[![Latest PyPI version](https://img.shields.io/pypi/v/jupyterhub-idle-culler?logo=pypi&logoColor=white)](https://pypi.python.org/pypi/jupyterhub-idle-culler)
[![GitHub](https://img.shields.io/badge/issue_tracking-github-blue?logo=github)](https://github.com/jupyterhub/jupyterhub-idle-culler/issues)
[![Discourse](https://img.shields.io/badge/help_forum-discourse-blue?logo=discourse)](https://discourse.jupyter.org/c/jupyterhub)
[![Gitter](https://img.shields.io/badge/social_chat-gitter-blue?logo=gitter)](https://gitter.im/jupyterhub/jupyterhub)

`jupyterhub-idle-culler` provides a JupyterHub service to identify and shut down idle or long-running Jupyter Notebook servers.
The exact actions performed are dependent on the used spawner for the Jupyter Notebook server (e.g. the default [LocalProcessSpawner](https://jupyterhub.readthedocs.io/en/stable/api/spawner.html#localprocessspawner>), [kubespawner](https://github.com/jupyterhub/kubespawner), or [dockerspawner](https://github.com/jupyterhub/dockerspawner)).
In addition, if explicitly requested, all users whose Jupyter Notebook servers have been shut down this way are deleted as JupyterHub users from the internal database. This neither affects the authentication method which continues to allow those users to log in nor does it delete persisted user data (e.g. stored in docker volumes for dockerspawner or in persisted volumes for kubespawner).

## Setup

### Installation

```bash
pip install jupyterhub-idle-culler
```

### As a hub managed service

In `jupyterhub_config.py`, add the following dictionary for the idle-culler
Service to the `c.JupyterHub.services` list:

```python
c.JupyterHub.services = [
    {
        'name': 'idle-culler',
        'admin': True,
        'command': [
            sys.executable,
            '-m', 'jupyterhub_idle_culler',
            '--timeout=3600'
        ],
    }
]
```

where:

- `'admin': True` indicates that the Service requires admin permissions so
  it can shut down arbitrary user notebooks, and
- `'command'` indicates that the Service will be managed by the Hub.

### As a standalone script

`jupyterhub-idle-culler` can also be run as a standalone script. It can
access the hub's api with a service token. The service token must have
admin privileges.

Generate an API token and store it in a `JUPYTERHUB_API_TOKEN` environment
variable. Then start `jupyterhub-idle-culler` manually

```bash
export JUPYTERHUB_API_TOKEN=$(jupyterhub token)
python3 -m jupyterhub-idle-culler [--timeout=900] [--url=http://localhost:8081/hub/api]
```

The command line interface also gives a quick overview of the different options for configuration.

```
  --concurrency                    Limit the number of concurrent requests made
                                   to the Hub.  Deleting a lot of users at the
                                   same time can slow down the Hub, so limit
                                   the number of API requests we have
                                   outstanding at any given time. (default 10)
  --cull-every                     The interval (in seconds) for checking for
                                   idle servers to cull. (default 0)
  --cull-users                     Cull users in addition to servers.  This is
                                   for use in temporary-user cases such as
                                   tmpnb. (default False)
  --internal-certs-location        The location of generated internal-ssl
                                   certificates (only needed with --ssl-
                                   enabled=true). (default internal-ssl)
  --max-age                        The maximum age (in seconds) of servers that
                                   should be culled even if they are active.
                                   (default 0)
  --remove-named-servers           Remove named servers in addition to stopping
                                   them.  This is useful for a BinderHub that
                                   uses authentication and named servers.
                                   (default False)
  --ssl-enabled                    Whether the Jupyter API endpoint has TLS
                                   enabled. (default False)
  --timeout                        The idle timeout (in seconds). (default 600)
  --url                            The JupyterHub API URL.
```

## Caveats

1. last_activity is not updated with high frequency, so cull timeout should be
   greater than the sum of:

   - single-user websocket ping interval (default: 30 seconds)
   - `JupyterHub.last_activity_interval` (default: 5 minutes)

2. The same `--timeout` and `--max-age` values are used to cull
   users and users' servers. If you want a different value for users and servers,
   you should add this script to the services list twice, just with different
   `name`s, different values, and one with the `--cull-users` option.

3. By default HTTP requests to the hub timeout after 60 seconds. This can be
   changed by setting the `JUPYTERHUB_REQUEST_TIMEOUT` environment variable.

## How it works

jupyterhub-idle-culler culls user servers using JupyterHub's REST API
([/users/{name}/server](https://jupyterhub.readthedocs.io/en/stable/_static/rest-api/index.html#operation--users--name--server-delete)
or
[/users/{name}/servers/{server_name}](https://jupyterhub.readthedocs.io/en/stable/_static/rest-api/index.html#operation--users--name--servers--server_name--delete)),
and makes the culling decisions based on its configuration and what JupyterHub
reports about the user servers via its REST API
[(/users)](https://jupyterhub.readthedocs.io/en/stable/_static/rest-api/index.html#path--users)
where user servers' `last_activity` is reported.

JupyterHub itself updates information about the user server's last activity at a
regular interval via the [`update_last_activity` function (Link to JupyterHub
version
1.4.1)](https://github.com/jupyterhub/jupyterhub/blob/1.4.1/jupyterhub/app.py#L2650).
The `update_last_activity` function then in turn collects and combines
information about activity both from the deployed JupyterHub's configured [Proxy
class](https://jupyterhub.readthedocs.io/en/stable/reference/proxy.html) and
configured [Spawner
class](https://jupyterhub.readthedocs.io/en/stable/reference/spawners.html).

The Proxy class, if it supports it, will return activity information related to
served network traffic between a web browser and the user server, while the
Spawner class will return activity information related to kernel's activity,
such as if a server is running code or similar.

The user server's kernel activity, which impacts the decisions made by
jupyterhub-idle-culler, can in turn be influenced by a kernel manager that
can be configured to cull kernels running within the user server. See the
[Jupyter Notebook server
documentation](https://jupyter-notebook.readthedocs.io/en/stable/config.html#options)
for more information about configuration options of relevance like these:

- `MappingKernelManager.cull_busy`
- `MappingKernelManager.cull_idle_timeout`
- `MappingKernelManager.cull_interval`
- `MappingKernelManager.cull_connected`

  Note that `cull_connected` probably makes sense to set to `True` if your
  JupyterHub users use a JupyterLab based user interface, see [this issue for
  more details](https://github.com/jupyterlab/jupyterlab/issues/6893).

Keep in mind that configuration of MappingKernelManager should be made on the
user server itself, for example via a `jupyter_notebook_config.py` file in
`/etc/jupyter` or `/usr/local/etc/jupyter` rather than where JupyterHub itself
is running.

Also note that a Jupyter Notebook server can shut itself down without without
intervention by the `jupyterhub-idle-culler` if
`NotebookApp.shutdown_no_activity_timeout` is configured.
