Mender artifact file format
===========================

File extension: `.mender`

The tree below describes the layout of files inside the main file, which is
hosted inside a standard tar archive.

Note that there are some restrictions on ordering of the files, described
in the "Ordering" section.


### version 3

```
-artifact.mender (tar format)
  |
  +---version
  |
  +---manifest
  |
  +---manifest.sig
  |
  +---manifest-augment
  |
  +---header.tar.gz (tar format)
  |    |
  |    +---header-info
  |    |
  |    +---scripts
  |    |    |
  |    |    +---State.Enter
  |    |    +---State.Leave
  |    |    +---State.Error
  |    |    `---<more scripts>
  |    |
  |    `---headers
  |         |
  |         +---0000
  |         |    |
  |         |    +---files
  |         |    |
  |         |    +---type-info
  |         |    |
  |         |    +---meta-data
  |         |
  |         +---0001
  |         |    |
  |         |    `---<more headers>
  |         |
  |         `---000n ...
  |
  +---header-augment.tar.gz (tar format)
  |    |
  |    +---header-info
  |    |
  |    `---headers
  |         |
  |         +---0000
  |         |    |
  |         |    +---files
  |         |    |
  |         |    +---type-info
  |         |    |
  |         |    +---meta-data
  |         |
  |         +---0001
  |         |    |
  |         |    `---<more headers>
  |         |
  |         `---000n ...
  |
  `---data
       |
       +---0000.tar.gz
       |    +--<image-file (ext4)>
       |    +--<binary delta, etc>
       |    `--...
       |
       +---0001.tar.gz
       |    +--<image-file (ext4)>
       |    +--<binary delta, etc>
       |    `--...
       |
       +---000n.tar.gz ...
            `--...
```


version
----

Format: JSON

Contains the below content exactly:

```
{
  "format": "mender",
  "version": 3
}
```

The `format` value is to confirm that this is indeed a Mender update file, and
the `version` value is a way to extend/change the format later if needed.


manifest
----

Format: text

Contains file checksums, formatted exactly like below:

```
1d0b820130ae028ce8a79b7e217fe505a765ac394718e795d454941487c53d32  data/0000/update.ext4
4d480539cdb23a4aee6330ff80673a5af92b7793eb1c57c4694532f96383b619  header.tar.gz
52c76ab66947278a897c2a6df8b4d77badfa343fec7ba3b2983c2ecbbb041a35  version
```

The manifest file contains checksums of compressed header, version and the
data files being a part of the artifact. The format matches the output of
`sha256sum` tool which is the sum and the name of the file separated by
the two spaces.


manifest.sig
----

Format: base64 encoded ecdsa or rsa signature

File containing the signature of `manifest`.

It is legal for an artifact not to have signature file.


manifest-augment
----

Format: text

Contains file checksums, formatted exactly like below:

```
4d480539cdb23a4aee6330ff80673a5af92b7793eb1c57c4694532f96383b619  header-augment.tar.gz
1d0b820130ae028ce8a79b7e217fe505a765ac394718e795d454941487c53d32  data/0000/update.delta
```

The manifest-augment file is the extension of manifest file and is needed only
for certain types of the updates.
It contains the checksums of the files which could change during the creation of the
artifact and therefore which can not be signed explicitly. In case of the
delta update this file will contain the checksum of the delta file (the actual
payload of the file being a part of the artifact) and the header-augment.tar.gz
file checksum.


header.tar.gz
-------------

Format: tar

A tar file that contains various header files.

Why is there a tar file inside a tar file? See the "Ordering" section.

### header-info

Format: JSON

`header-info` must be the first file within `header.tar.gz`. Its content is:

```
{
    "updates": [
        {
            "type": "rootfs-image"
        },
        {
            "type": "delta-image"
        }
    ],
    "artifact_provides": {
        "artifact_name": "release-2",
        "artifact_group": "fix",
        "update_types_supported": [
            "rootfs-image",
            "delta-update"
        ]
    },
    "artifact_depends": {
        "artifact_name": "release-1",
        "device_type": [
            "vexpress-qemu",
             "beaglebone"
        ]
    }
}
```

The `updates` list is a list of all the updates contained within the
artifact. The intention of having multiple updates is to allow proxy based
updates to deploy to several different hosts at the same time. However, for
updates downloaded to single devices, there will usually be only one.

`type` is the type of update contained within the image. At the moment there is
only `rootfs-image`, but there may be others in the future, like `docker-image`
or something package based.

The remaining entries in `header.tar.gz` are then organized in buckets under
`headers/xxxx` folders, where `xxxx` are four digits, starting from zero, and
corresponding to each element `updates` inside `header-info`, in order. The
following sub sections define each field under each such bucket.


#### artifact_depends

The `artifact_depends` contains a set of parameters that the current artifact
depends on. It can contain zero or more key/value pairs (in most cases at least
`device_type` should be present though).

The given artifact will be installed, only if the device itself
and the artifact currently installed on the device are providing a full set
of matching parameters. The complete list contains following parameters:

* `artifact_name` is the name of the artifact currently installed on the device
* `device_type` is the type of the device (see `device_provides` below)
* `artifact_group` is the group of the artifacts current artifact belongs to
* `update_types_supported` is the list of the update types Mender agent installed
on the device can handle
* `artifact_versions_supported` is the list of the versions of the artifacts
agent installed on the device can handle


#### artifact_provides

The `artifact_provides` is a set of global parameters given artifact provides.
For the detailed information see the description of the given parameter below.

* `artifact_name` is the name of the artifact
* `artifact_group` is the name of the group of artifacts given artifact
belongs to
* `update_types_supported` is the list of all the update types given Mender
client can install

#### device_provides

There is also a set of parameters that are provided by the device itself,
which are not a part of the artifact. Those are the values, that the device
itself can read and send to the Mender server when needed. The full list of
`device_provides` is as follows:

* `device_type` is the current device type


### files

Format: JSON

Contains a JSON list of file names that make up the payload for this update (the
image file / package file / etc.), listed as bare file names. There may be one
or multiple files listed.  For example:

```
{ "files" : ["core-image-minimal-201608110900.ext4", "core-image-base-201608110900.ext4"]}
```

### type-info

Format: JSON

A file that provides information about the type of package contained within the
tar file. The first and the only required entry is the type of the update
corresponding to the type in `header-info` file.
It can also contain some additional parameters extending or modifying the global
`artifact_provides` set of parameters specific for a given update type.

```
{
  "type": "rootfs-image"
  "artifact_provides": {
      "rootfs_image_checksum": "4d480539cdb23a4aee6330ff80673a5af92b7793eb1c57c4694532f96383b619"
  },
  "artifact_depends": {
      "rootfs_image_checksum": "4d480539cdb23a4aee6330ff80673a5af92b7793eb1c57c4694532f96383b619"
  },
}
```

#### artifact_provides

As an opposite to the list of global `artifact_provides` being a part of
`header-info` file, the `artifact_provides` section in the `type-info` file
is a set of parameters specific for a given update type.

The list of currently supported parameters is as follows:

* `rootfs_image_checksum` is the checksum of the image contained in the artifact

#### artifact_depends

The `artifact_depends` section in the `type-info` file is a set of parameters
specific for a given update type. The list of currently supported
parameters is as follows:

* `rootfs_image_checksum` is the checksum of the image that needs to be installed
on the device before current artifact can be installed


### meta-data

Format: JSON

Meta data about the image. This depends on the `type` in `header-info`. For
`rootfs-image` there are no additional information needed and the file might
be empty.

For other package types this file can contain for example number of files in the
`data` directory, if the update contains more than one. Or it can contain
network address(es) and credentials if Mender is to do a proxy update.


### scripts

Format: Directory containing script files.

Any script, or even the whole directory, can be missing if there are no scripts
of that type, or at all.

Each script corresponds to a Mender state according to the script API, and
consists of up to three events, `Enter`, `Leave` and `Error`, which are executed
before the state is entered, and before leaving the state for
another one, respectively.

The complete script API consists of the following scripts:

* `(Idle.Enter)`
* `(Idle.Leave)`
* `(Idle.Error)`
* `(Sync.Enter)`
* `(Sync.Leave)`
* `(Sync.Error)`
* `(Download.Enter)`
* `(Download.Leave)`
* `(Download.Error)`
* `ArtifactInstall.Enter`
* `ArtifactInstall.Leave`
* `ArtifactInstall.Error`
* `ArtifactReboot.Enter`
* `ArtifactReboot.Leave`
* `ArtifactReboot.Error`
* `ArtifactCommit.Enter`
* `ArtifactCommit.Leave`
* `ArtifactCommit.Error`
* `ArtifactRollback.Enter`
* `ArtifactRollback.Leave`
* `ArtifactRollback.Error`
* `ArtifactRollbackReboot.Enter`
* `ArtifactRollbackReboot.Leave`
* `ArtifactRollbackReboot.Error`
* `ArtifactFailure.Enter`
* `ArtifactFailure.Leave`
*  **IMPORTANT** there is no `ArtifactFailure.Error` state script support

States in parentheses are states that are supported as scripts on stored on the
filesystem, but are not included in the artifact itself.

For more information about the script and state API, see the official Mender
documentation.


header-augment.tar.gz
-------------

Format: tar

This file is complementing the information contained in the header.tar.gz.
It can have the same structure as header.tar.gz, but for security reasons
(this file is not signed) only certain files and parameters are allowed.

These files and attributes are allowed:

* `header-info` file with one list attribute:
  ```
  {
    "updates": [
        {
            "type": "rootfs-image"
        },
        {
            "type": "delta-image"
        }
    ]
  }
  ```
  The `updates` attribute is expected to be in the same order as the original in
  `header.tar.gz`, and will override it.

* `type-info` file with `artifact_depends` and `rootfs_image_checksum`:
  ```
  {
    "type": "rootfs-image"
    "artifact_depends": {
        "rootfs_image_checksum": "4d480539cdb23a4aee6330ff80673a5af92b7793eb1c57c4694532f96383b619"
    },
  }
  ```

At the moment ONLY a `type-info` file is allowed which can contain only
`artifact_depends` with one field: `rootfs_image_checksum` parameters and type of the update.


data
----

Format: Directory containing image files.

All files listed in the tar archive under the `data` directory must be after all
other files. If any non-`data` file is found after a `data` file, this will
cause the update to immediately fail.

The rationale behind failing if `data` files are not last is that the client
should know everything that is possible about the update *before* the payload
arrives. Receiving this knowledge later might be at a point where it's too late
to apply it, hence this precaution.

It is legal for an update file to not contain any `data` files at all. In such
cases it is expected that the update type in question will receive the update
payload by using alternative means, such as providing a download link in
`type-info`.

Each file in the `data` folder should be a file of the format `xxxx.tar.gz`,
where `xxxx` are four digits corresponding to each entry in the `updates` list
in `header-info`, in order. Each file inside the `xxxx.tar.gz` archive should be
a file name corresponding exactly to a filename from the `files` header under
the corresponding header bucket. If the list of files found inside `xxxx.tar.gz`
is in any way different from the files listed in `files`, an error should be
produced and the update should fail.


Ordering
========

Some ordering rules are enforced on the artifact tar file. For the outer tar
file:

| File/Directory            | Ordering rule                  |
|---------------------------|--------------------------------|
| `version`                 | First in `.mender` tar archive |
| `manifest`                | After `version`                |
| `manifest.sig`            | Optional after `manifest`      |
| `manifest-augment`        | Optional after `manifest.sig`  |
| `header.tar.gz`           | After all manifest files       |
| `header-augment.tar.gz`   | Optional after `header.tar.gz` |
| `data`                    | After `header.tar.gz`          |

For the embedded `header.tar.gz` file:

| File/Directory  | Ordering rule                 |
|-----------------|-------------------------------|
| `header-info`   | First in `header.tar.gz` file |
| `scripts`       | Optional after `header-info`  |
| `headers`       | After `scripts`               |
| `files`         | First in every `xxxx` bucket  |
| `type-info`     | After `files`                 |
| `meta-data`     | After `type-info`             |

The fact that many files/directories inside `header.tar.gz` have ambiguous rules
(`checksums` can be before or after `signatures`) implies that the order is not
strictly enforced for these entries. The client is expected to cache what it
needs during reading of `header.tar.gz` even if reading out-of-order, and then
assembling the pieces when `header.tar.gz` has been completely read.

So why do we have the `header.tar.gz` tar file inside another tar file? The reason
is that order is important, and the files in the `data` directory need to come
last in order for the download to be efficient (by the time the main image
arrives, the client must already know everything in the header). Since the
number of files in the header can vary depending on what scripts are used and
whether signatures are enabled for example, it is better to group these together
in one indivisible unit which is the `header.tar.gz` file, rather than being
"surprised" by a signature file that arrives after the data file. All this is
not a problem as long as the mender tool is used to manipulate the
`artifact.mender` file, but if anyone does a custom modification using regular
tar, this is important.


Compression
===========

All file tree components ending in `.gz` in the tree displayed above should be
compressed, and the suffix corresponds to the compression method.
