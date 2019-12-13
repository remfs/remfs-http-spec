# Introduction

remFS is a network filesystem which is accessed over HTTP. The core design
principle comes from this Alan Kay quote:

**"Simple things should be simple, complex things should be possible." - Alan Kay**


# Features

remFS servers choose from discrete sets of "features" to implement. These
features are designed to be complimentary but independent, and it's likely that
additional features will be made available over time (such as auth). The
current features are **read**, **write**, **events**, and **perf**. read,
write, and events all add functionality. In theory, anything you would want to
do with a filesystem can be done with only these features. In practice, perf is
necessary in order to achieve good performance for many real-world
applications. The corresponding functionality for each feature is listed in the
following table:

<table>
  <tbody>
    <tr>
      <th>read</th>
    </tr>
    <tr>
      <td>
        <ul>
          <li>
            GET URLs ending in /remfs.json either return metadata or 404.
          </li>
          <li>
            GET on files returns file contents.
          </li>
          <li>
            (Recommended) Support byte range requests for files .
          </li>
          <li>
            (Recommended) GET on directories returns simple HTML listings for
            navigation in web browsers.
          </li>
        </ul>
      </td>
    </tr>
    <tr>
      <th>write</th>
    </tr>
    <tr>
      <td>
        <ul>
          <li>
            PUT creates a new file, or directory if query parameter type=dir
          </li>
          <li>
            PATCH modifies an existing file, starting at the byte index
            indicated by the header RemFS-Offset, and copying Content-Length
            bytes.
          </li>
          <li>
            DELETE removes the item from the filesystem.
          </li>
          <li>
            GET/HEAD must return exact number of bytes on disk. This can be
            used with PATCH to resume failed uploads.
          </li>
        </ul>
      </td>
    </tr>
    <tr>
      <th>events</th>
    </tr>
    <tr>
      <td>
        <ul>
          <li>
            All paths support the query parameter events=true, which will
            return a server-sent events response.
          </li>
          <li>
            All mutation events (create, update, delete, etc) will result in
            an event being broadcast to every subscriber on the mutated path,
            and any parent paths of the mutated path.
          </li>
        </ul>
      </td>
    </tr>
    <tr>
      <th>perf</th>
    </tr>
    <tr>
      <td>
        <ul>
          <li>
            All file GETs must support byte range request queries.
          </li>
          <li>
            All previous requests can alternatively be sent as JSON POST
            requests, with Content-Type set to text/plain. This allows for
            avoiding CORs preflight requests, which is important for
            latency-sensitive applications. Implementing servers must make sure
            to implement proper cross-origin security measures.
          </li>
          <li>
            Implement JSON POST move request, for moving/renaming files and
            directories.
          </li>
          <li>
            Implement JSON POST concat request, for combining multiple files
            into a new single file. This is useful for initiating multiple
            uploads to temporary files for performance reasons, then collecting
            the parts into the final file.
          </li>
        </ul>
      </td>
    </tr>
  </tbody>
</table>

## Additional Information

It is recommended for remFS servers to indicate which features each file or
directory supports, like this:

```json
{
  "features": {
    "read": true,
    "write": true,
    "events": true,
    "perf": false,
  },
  "type": "dir",
  "children": {
  }
}
```

This information is not mandatory. If missing, clients can simply attempt
the desired request, and the server can indicate an error if it's not
supported.

If the information is provided in a given directory of the filesystem (for
example at the root), it can be omitted for any descendants of that directory.
Features are assumed to be the same down the tree until new values are
encountered. This allows for implementing rudimentary permissions
functionality.

### read

The read feature is designed in such a way that read-only remFS servers can
implemented using nothing but an off-the-shelf static file HTTP server and a
hand-written remfs.json file. For example, if you had the following directory
structure:

```
d1/
  f1.txt
  f2.mp3
  d2/
    f3.pdf
    f4.md
```

You could create this remfs.json file:

```json
{
  "type": "dir",
  "children": {
    "f1.txt": {
      "type": "file"
    },
    "f2.mp3": {
      "type": "file"
    },
    "d2": {
     "type": "dir",
     "children": {
       "f3.pdf": {
         "type": "file"
       },
       "f4.md": {
         "type": "file"
       }
     }
    }
  }
}
```

Place that file in `d1`, and start a static web server in that directory. The
server will be browseable as a read-only remFS filesystem.

### perf

Example JSON POST move:

```json
{
  "method": "move",
  "srcPath": "/path/to/item",
  "dstPath": "/path/to/destination"
}
```

Example JSON POST concat:

```json
{
  "method": "concat",
  "srcPaths": [
    "/path/to/file1",
    "/path/to/file2",
    "/path/to/file3"
  ],
  "dstPath": "/path/to/destination/file"
}
```
