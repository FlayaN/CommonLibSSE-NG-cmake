# USAGE:

```cmake
add_simple_commonlibsse_ng_plugin(
  # The plugin's license, empty by default.
  LICENSE <string>

  # The path to the src folder, defaults to src
  SRC_FOLDER <string>

  ... also supports all options that add_commonlibsse_plugin func supports!
)
```
# List of all options that add_commonlibsse_plugin supports for reference

```cmake
add_commonlibsse_plugin(<target>
    # The plugin's name, defaults to target.
    NAME <string>

    # The plugin's author, empty by default.
    AUTHOR <string>

    # The support email address, empty by default.
    EMAIL <string>

    # The plugin version number, defaults to ${PROJECT_VERSION}.
    VERSION <version number>

    # Indicates the plugin is compatible with all runtimes via address library. This is the default if no
    # other compatibilility mode is specified. Can be used with USE_SIGNATURE_SCANNING but not
    # COMPATIBLE_RUNTIMES.
    USE_ADDRESS_LIBRARY

    # Indicates the plugin is compatible with all runtimes via signature scanning.  Can be used with
    # USE_ADDRESS_LIBRARY but not COMPATIBLE_RUNTIMES.
    USE_SIGNATURE_SCANNING

    # If specificed it will set plugin struct compatibility to ::Dependent otherwise ::Independent
    # default ::Independent
    STRUCT_DEPENDENT

    # List of up to 16 Skyrim versions the plugin is compatible with. Cannot be used with
    # USE_ADDRESS_LIBRARY or USE_SIGNATURE_SCANNING.
    COMPATIBLE_RUNTIMES <version number> [<version number>...]

    # The minimum SKSE version to support; defaults to 0, and recommended by SKSE project to be left
    # 0.
    MINIMUM_SKSE_VERSION <version number>

    # Omit from all targets, same as used with add_library.
    EXCLUDE_FROM_ALL

    # List of the sources to include in the target, as would be the parameters to add_library.
    SOURCES <path> [<path> ...]
)
```
