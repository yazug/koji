=========================
How to write Koji plugins
=========================


Writing Koji plugins
====================

Depending on what you are trying to do, there are different ways to
write a Koji plugin.

Each is described in this file, by use case.

Adding new task types
---------------------

Koji can do several things, for example build RPMs, or live CDs. Those
are types of tasks which Koji knows about.

If you need to do something which Koji does not know yet how to do, you
could create a Koji Builder plugin.

Such a plugin would minimally look like this:

::

    from koji.tasks import BaseTaskHandler

    class MyTask(BaseTaskHandler):
        Methods = ['mytask']
        _taskWeight = 2.0

        def handler(self, arg1, arg2, kwarg1=None):
            self.logger.debug("Running my task...")

            # Here is where you actually do something

A few explanations on what goes on here:

-  Your task needs to inherit from ``koji.tasks.BaseTaskHandler``
-  Your task must have a ``Methods`` attribute, which is a list of the
   method names your task can handle.
-  You can specify the weight of your task with the ``_taskWeight``
   attribute. The more intensive (CPU, IO, ...) your task is, the higher
   this number should be.
-  The task object has a ``logger`` attribute, which is a Python logger
   with the usual ``debug``, ``info``, ``warning`` and ``error``
   methods. The messages you send with it will end up in the Koji
   Builder logs (``kojid.log``)
-  Your task must have a ``handler()`` method. That is the method Koji
   will call to run your task. It is the method that should actually do
   what you need. It can have as many positional and named arguments as
   you want.

Save your plugin as e.g ``mytask.py``, then install it in the Koji
Builder plugins folder: ``/usr/lib/koji-builder-plugins/``

Finally, edit the Koji Builder config file, ``/etc/kojid/kojid.conf``:

::

    # A space-separated list of plugins to enable
    plugins = mytask

Restart the Koji Builder service, and your plugin will be enabled.

You can try running a task from your new task type with the
command-line:

::

    $ koji make-task mytask arg1 arg2 kwarg1

Exporting new API methods over XMLRPC
-------------------------------------

Koji clients talk to the Koji Hub via an XMLRPC API.

It is sometimes desirable to add to that API, so that clients can
request things Koji does not expose right now.

Such a plugin would minimally look like this:

::

    def mymethod(arg1, arg2, kwarg1=None):
        # Here is where you actually do something

    mymethod.exported = True

A few explanations on what goes on here:

-  Your plugin is just a method, with whatever positional and/or named
   arguments you need.
-  You must export your method by setting its ``exported`` attribute to
   ``True``
-  The ``context.session.assertPerm()`` is how you ensure that the

Save your plugin as e.g ``mymethod.py``, then install it in the Koji Hub
plugins folder: ``/usr/lib/koji-hub-plugins/``

Finally, edit the Koji Hub config file, ``/etc/koji-hub/hub.conf``:

::

    # A space-separated list of plugins to enable
    Plugins = mymethod

Restart the Koji Hub service, and your plugin will be enabled.

You can try calling the new XMLRPC API with the Python client library:

::

    >>> import koji
    >>> session = koji.ClientSession("http://koji/example.org/kojihub")
    >>> session.mymethod(arg1, arg2, kwarg1='some value')

Ensuring the user has the required permissions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you want your new XMLRPC API to require specific permissions from the
user, all you need to do is add the following to your method:

::

    from koji.context import context

    def mymethod(arg1, arg2, kwarg1=None):
        context.session.assertPerm("admin")

        # Here is where you actually do something

    mymethod.exported = True

In the example above, Koji will ensure that the user is an
administrator. You could of course create your own permission, and check
for that.

Running code automatically triggered on events
----------------------------------------------

You might want to run something automatically when something else
happens in Koji.

A typical example is to automatically sign a package right after a build
finished. Another would be to send a notification to a message bus after
any kind of event.

This can be achieved with a plugin, which would look minimally as
follows:

::

    from koji.plugin import callback

    @callback('preTag', 'postTag')
    def mycallback(cbtype, tag, build, user, force=False):
        # Here is where you actually do something

A few explanations on what goes on here:

-  The ``@callback`` decorator allows you to declare which events should
   trigger your function. You can pass as many as you want. For a list
   of supported events, see ``koji/plugins.py``.
-  The arguments of the function depend on the event you subscribed to.
   As a result, you need to know how it will be called by Koji. You
   probably should use ``*kwargs`` to be safe. You can see how callbacks
   are called in the ``hub/kojihub.py`` file, search for calls of the
   ``run_callbacks`` function.

Save your plugin as e.g ``mycallback.py``, then install it in the Koji
Hub plugins folder: ``/usr/lib/koji-hub-plugins``

Finally, edit the Koji Hub config file, ``/etc/koji-hub/hub.conf``:

::

    # A space-separated list of plugins to enable
    Plugins = mycallback

Restart the Koji Hub service, and your plugin will be enabled.

You can try triggering your callback plugin with the command-line. For
example, if you registered a callback for the ``postTag`` event, try
tagging a build:

::

    $ koji tag-build mytag mypkg-1.0-1


.. _plugin-cli-command:

New command for CLI
-------------------

When you add new XMLRPC call or just wanted to do some more complicated
things with API, you can benefit from writing a new command for CLI.

Most simple command would look like this:

::

    from koji.plugin import export_cli

    @export_cli
    def anon_handle_echo(options, session, args):
        usage = _("usage: %prog echo <message>")
        parser = OptionParser(usage=usage)
        (opts, args) = parser.parse_args(args)
        print(args[0])

`@export_cli` is decorator which will register plugin's command with name
derived from name of the function. Function name needs to follow one rule: It
has to start with `anon_handle_` or `handle_`. Rest of the name is name of
the command. First one will not authenticate against hub (user can still override
this behaviour with `--force-auth` or `--mine` options where it is relevant)
- it is simply same as using `--noath` option. Second variant doesn't presume
  anything about authentication.

Compared to this simple example is real usage with API calls. Koji provides
some important functions via client library `koji_cli.lib` which can be
imported and used inside command's function. Feel free to examine, what is
provided there. Some notable examples are:

 * `activate_session(session, options)` - It is needed to authenticate
   against hub. Both parameters are same as those passed to handler.
 * `watch_tasks(session, tasklist, quiet=False, poll_interval=60)` - It is
   the same function used e.g. in `build` command for waiting for spawned
   tasks.
 * `list_task_output_all_volumes(session, task_id)` - wrapper function for
   `listTaskOutput` with different versions of hub.

Final command has to be saved in python system-wide library path - e.g. in
`/usr/lib/python3.4/site-packages/koji_cli_plugins`. Filename doesn't matter
as all files in this directory are searched for `@export_cli` macros. Note,
that python 3 variant of CLI is looking to different directory than python 2
one.

CLI plugins structure will be extended (made configurable and allowing more
than just adding commands - e.g. own authentication methods, etc.) in future.

Pull requests
^^^^^^^^^^^^^

These plugins have to be written in python 2.6+/3.x compatible way. We are
using `six` library to support this, so we will also prefer pull requests
written this way. CLI (and client library) is meant to be fully compatible
with python 3 from koji 1.13.

Tests are also recommended for PR. For example one see
`tests/test_plugins/test_runroot_cli.py`.
