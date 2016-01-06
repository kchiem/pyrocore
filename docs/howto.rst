Tips & How-Tos
==============

Dumping Items as a JSON Array
-----------------------------

If you want to access rTorrent item data in machine readable form via ``rtcontrol``,
you can use its ``--json`` option and feed the output into another script parsing
the JSON data for further processing.

Here's an example:

.. code-block:: shell

    $ rtcontrol --json -qo name,is_ghost,directory,fno foo
    [
      {
        "directory": "/var/torrent/load/foo",
        "fno": 1,
        "is_ghost": false,
        "name": "foo"
      }
    ]


Working With Several rTorrent Instances
---------------------------------------

Both ``rtcontrol`` and ``rtxmlrpc`` read the existing rTorrent configuration
to extract some settings, so that you don't need to maintain them twice – most
importantly the details of the XMLRPC connection. That is why ``config.ini``
has the ``rtorrent_rc`` setting, and changing that is the key to select
a different instance you have running.

Just pass the option ``-D rtorrent_rc=PATH_TO/rtorrent.rc`` to either
``rtcontrol`` or ``rtxmlrpc``, to read the configuration of another instance
than the default one. For convenient use on the command line, you can add
shell aliases to you profile.


Using Tags or Flag Files to Control Item Processing
---------------------------------------------------

If you want to perform some actions on download items exactly once,
you can use tags or flag files to mark them as handled.
The basic pattern works like this:

.. code-block:: shell

    #! /usr/bin/env bash
    guard="handled"
    …

    rtcontrol --from-view complete -qohash tagged=\!$guard | \
    while read hash; do
        …

        # Mark item as handled
        rtcontrol -q hash=$hash --tag "$guard" --flush --yes --cron
    done

A variant of this is to use a flag file in the download's directory –
such a file can be created and checked by simply poking the file system, which
can have advantages in some situations. To check for the existance
of that file, add a custom field to your ``config.py`` as follows::

    def is_synced(obj):
        "Check for .synced file."
        pathname = obj.path
        if pathname and os.path.isdir(pathname):
            return os.path.exists(os.path.join(pathname, '.synced'))
        else:
            return False if pathname else None

    yield engine.DynamicField(engine.untyped, "is_synced", "does download have a .synced flag file?",
        matcher=matching.BoolFilter, accessor=is_synced,
        formatter=lambda val: "SYNC" if val else "????" if val is None else "!SYN")

The condition ``is_synced=no`` is then used instead of the ``tagged`` one in the bash snippet above,
and setting the flag is a simple ``touch``. Add a ``rsync`` call to the ``while`` loop in the example
and you have a cron job that can be used to transfer completed items to another host *exactly once*.
Note that this only works for multi-file items, since a data directory is assumed –
supporting single-file items is left as an exercise for the reader.
See :ref:`CustomFields` for more details regarding custom fields.