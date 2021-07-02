# Sailjail

Sailfish OS application sandboxes are launched through Sailjail that is a thin Firejail wrapper. The Firejail documentation can be found from [here](https://firejail.wordpress.com/).

In this document we are following terminology defined by the Firejail.

## Application permissions

Application [permissions](https://github.com/sailfishos/sailjail-permissions#sailfish-os-application-sandboxing-and-permissions) defined in Sailjail-permissions are Firejail profiles. Application developer defines set of required permissions in the application desktop file.

Application desktop file changes are described in the [Sailjail Permissions documentation](https://github.com/sailfishos/sailjail-permissions#enable-sandboxing-for-an-application).

Sailjail parses the permissions and builds Firejail command line arguments out of the requested permissions.

## Application data structure

There are three directories that are whitelisted automatically by the Sailjail based on the
desktop file definition.

    $HOME/.local/share/<OrganizationName>/<ApplicationName>
    $HOME/.cache/<OrganizationName>/<ApplicationName>
    $HOME/.config/<OrganizationName>/<ApplicationName>


Default user data directories such as [*Pictures*, *Videos*, *Music*, *Documents*](https://www.freedesktop.org/wiki/Software/xdg-user-dirs/) are not whitelisted by default rather application developer needs to request access with predefined [Permissions](https://github.com/sailfishos/sailjail-permissions#permissions).

## Implicit D-Bus user service ownership

Based on [*OrganizationName* and
*ApplicationName* values](https://github.com/sailfishos/sailjail-permissions#desktop-file-changes)
application is granted rights to own a D-Bus service name on user session bus.

Example desktop file

    [Desktop Entry]
    Type=Application
    Name=My Application
    Icon=my-app-icon
    Exec=/usr/bin/org.foobar.MyApp

    [X-Sailjail]
    Permissions=Internet;Pictures
    OrganizationName=org.foobar
    ApplicationName=MyApp

With above example application desktop file Firejail command line arguments contain implicitly
**--dbus-user.own=org.foobar.MyApp** when launched through Sailjail.
When application's executable matches desktop file name, it is enough to define it in Exec value. If
the names differ you may define the desktop file with **sailjail** command's **-p** argument:

    Exec=/usr/bin/sailjail -p foobar-myapp.desktop -- /usr/bin/org.foobar.MyApp

Use of absolute paths is required, except for the desktop file which must be located in
_/usr/share/applications_ or _/etc/sailjail/applications_.

## Homescreen integration
The Sailjail package contains sailjail-plugin-devel subpackage that provides interfaces for building homescreen integration plugins. The idea is that plugin adds launching hooks and Sailjail triggers a hook to confirm the launch. The plugin in turn replies by denying or accepting the application launch. In case the plugin replies that permissions are not granted (i.e. are denied) Sailjail refuses to start the application.

Sailfish OS implements a homescreen integration plugin. Application permission are requested from user upon first application launch and accepted permissions are stored to root readable directory under */var/lib/sailjail-homescreen/\<uid\>/\<full-desktop-file-path\>/X-Sailjail*. Subsequent application launches do not re-trigger the permission request.

When an application is upgraded and new permissions are introduced, permissions are requested again from the user. If permissions are removed from the application during upgrade, already approved permissions are considered sufficient and an intersection of permissions defined in the application desktop file and approved permissions are applied to the application.

When going forward we are working towards more dynamic permission management system. Requested application permissions are visible in Settings -> Apps -> Application when application is sandboxed.

## Sailfish OS specific changes to Firejail

### Handling privileged user data

In Sailfish OS portion of user data is considered private and although it resides within user's home
directory the files / directories are owned by _privileged_ user / group and only select
applications are granted access via executing them with effective group id set to _privileged_. For
sandboxed applications _private-data \<directory\>_ Firejail rule or command line option is used to
provide more fine grained access to privileged data.

Also noteworthy: sandboxed applications are executed with _no-new-privs_ set. As this makes it
impossible to have effective gid that differs from real gid, privileged applications are running
with real group id set to _privileged_.

### General fixes that might be eligible for upstreaming

- Rule templating. Sailjail defines keywords used in Firejail profiles that Firejail substitutes
  automatically while parsing the rules.
- Loading profiles only after parsing arguments instead of loading while parsing.

## Debugging sandboxed applications

See [Application debugging instructions](APPDEBUG.md).
