# About ALTO-server project

## Prerequisites

In current approach we require the following extra python libraries (some other
packages such as `configparser` often comes with a python distribution)

- [`bottle.py`][bottle.py],
- [`mimeparse`][mimeparse],
- [`SubnetTree`][subnettree]

Also since the packages in [backends](backends) and [plugins](plugins) are
required using the example configuration file, the path must be added to the
python path before running the server. It can be done by putting the following
line in files such as `.bashrc`:

~~~bash
export PYTHONPATH=$PATH_TO_ALTO_SERVER:$PATH_TO_BACKENDS:$PATH_TO_PLUGINS:$PYTHONPATH
~~~

## Architecture

See [architecture](docs/architecture.png).

Each instance **MAY** have multiple roles. The only one required is the *HTTP
service provider* which is named `palto.Backend` currently.

Only if the backend instance wants to be included in the IRD provided by palto
server, the `ird` section in the corresponding configuration file **CAN** and
**MUST** be provided. In that circumstance, the instance has to implement the
`BasicIRDResource` interface. The *mountpoint* option in *ird* section specifies
where the resource wants to register itself.

### Front-end

We will be using [bottle][bottle.py] to provide RESTful service.

### Backend

There are currently two backend types to be supported:

- Static file backend: reading data from a static file
- Open Daylight backend: reading data from an ODL controller

#### Implementing a customized backend

To write a customized backend, the module should contain the following method:

~~~
create_instance(resource-id, backend-config, environment)
~~~

The global configuration is now stored as `environment['config']`.

## Usage

~~~
python -m palto.server -c examples/palto.conf
~~~

## Debugging

BasicIRD and some standard handlers are tested by:

~~~
python -m palto.rfc7285.irdhandler
~~~

[bottle.py]: http://bottlepy.org/
[mimeparse]: https://github.com/dbtsai/python-mimeparse
[subnettree]: https://github.com/bro/pysubnettree

# About Advanced `PaltoServer` and `PaltoManager`

A progress to enhance the management ability of `palto` is on-going. A new
`PaltoServer` and `PaltoManager` are implemented. Trying them out with:

~~~bash
python -m palto.paltoserver

python -m palto.paltomanager
~~~

There are currently 4 basic operations on `PaltoManager`:

- `add-backend`/`remove-backend`: add/remove a backend from the managed
  `PaltoServer` by making a **POST** request to `/admin/<operation>/<name>`
- `install`/`uninstall`: install/uninstall a bottle plugin from either the
  `PaltoManager` or the `PaltoServer`, depending on the `target` parameter,
  by making a **POST** request to `/admin/<operation>/<target>/<name>`

The content of the **POST** request would be treated as a configuration file.
All the operations would require a `provider` option in the `basic` section.
The provider module must come with a `create_instance` method which takes the
`name`, `config`(parsed from the request) and `global_config`(initialized when
the manager is launched and maintained by the manager).

And you can use the following commands to test them out:

~~~bash
# for paltoserver

## should get 501
curl -D - -X GET http://localhost:3400/test

## should get 200
curl -D - -X GET http://localhost:3400/get_baseurl

# for paltomanager

## should get 404
curl -D -X GET http://localhost:3400/alto/test_sf_networkmap

## should get 200
curl -D - -X POST -u test:test \
      --data-binary @examples/input/test_staticfile.conf \
      http://localhost:3400/admin/add-backend/test_sf_networkmap

## should get 200
curl -D - -X GET http://localhost:3400/alto/test_sf_networkmap

## should get 200
curl -D - -X POST -u test:test \
      http://localhost:3400/admin/remove-backend/test_sf_networkmap

## should get 200
curl -D - -X POST -u test:test \
      --data-binary @examples/input/test_plugin.conf \
      http://localhost:3400/admin/install/paltoserver/test_plugin

## should get 200 and see some output in the server console
curl -D - -X GET http://localhost:3400/alto/test_sf_networkmap

## should get 200
curl -D - -X POST -u test:test \
      http://localhost:3400/admin/uninstall/paltoserver/test_plugin

## Install AutoIRDPlugin
## should get 200
curl -D - -X POST -u test:test \
      --data-binary @examples/input/test_autoird.conf \
      http://localhost:3400/admin/install/paltomanager/autoird

## should get 200 and see the generated root IRD
curl -D - -X GET http://localhost:3400/alto/

## should get 200 and see no output in the server console
curl -D - -X GET http://localhost:3400/alto/test_sf_networkmap

## should get 404 again
curl -D -X GET http://localhost:3400/alto/test_sf_networkmap

~~~

# About Static File Backend

The support for network map is implemented. The example data file can be found
in `examples/static_file/networkmap.json`.

## Testing static file backend

Run the following command in one terminal to start the server:

~~~
python -m palto.server -c examples/palto.conf
~~~

Then run the following command in another terminal to see the result:

~~~
curl -D - -X GET http://localhost:3400/test_sf_networkmap
curl -D - -X GET http://localhost:3400/test_sf_costmap
~~~

You can also run the following command to see the generated ird:

~~~
curl -D - -X GET http://localhost:3400/
~~~

# About the XXXLite

A simple implementation for several services.

See [ECSLite](backends/paltolite/README.md)

# About AutoIRDPlugin

A simple IRD generation plugin for palto.

See [AutoIRDPlugin](plugins/autoird/README.md) for details.

# Features in the Future

- Improve the code structure for server/backend/frontend (on-going)
- Management interface for palto (on-going)
- Dependency management and tracing (especially for ODL-backend)
- Tag support
