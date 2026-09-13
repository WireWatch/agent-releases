# Install the Wirewatch agent from the release binary

**The install steps for this route now live at
[docs.wirewatch.io/agents/binary/](https://docs.wirewatch.io/agents/binary/).** That page is built
from the same repository as the app's enrolment screen and moves with each release; this file used
to carry a second copy of it, which is a copy that drifts.

Read [the route overview](https://docs.wirewatch.io/agents/) first if you do not have a token yet —
it covers getting one, picking a route, and confirming the agent came up.

The notes below are kept here because they are not on the docs site.

## If something goes wrong

**`no file was verified`** from the checksum step means the archive never downloaded — usually a
wrong architecture or a network failure mid-transfer. Nothing was installed and your token is
unspent; fix the cause and paste the block again.

This is worth knowing because `curl` will not tell you: with several URLs it reports only the *last*
transfer's status, so a 404 on the archive can still exit `0`. The checksum step is what catches it.

**`--config-dir` must match in both places.** It appears in the enrolment line and in the unit's
`ExecStart`, and they must name the same directory. Enrolling into one and starting from another
spends the single-use token and leaves the service with nothing to resume from. The directory is
created `0700` with the key at `0600`, owned by root, since both steps run under `sudo`.

**A signature that does not verify means the file is not ours — do not run it.** See the
[verification instructions](./README.md#verify-what-you-downloaded) for how to check any release
asset on its own.
