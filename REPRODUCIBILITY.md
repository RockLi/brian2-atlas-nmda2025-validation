# Reproducibility scope

This exchange package supports two checks:

1. Fetch the immutable upstream repository and inspect the exact Brian2 and
   NEST scripts at commit `68e6dd970cfc6bab26459fcb8c34ee0f16560d9e`.
2. Verify every curated file against the release manifest with
   `shasum -a 256 -c MANIFEST.sha256`.

Repeating the backend measurements requires access to the tested Brian2-Atlas
build and the complete private experiment archive. The package therefore
presents concise validation results rather than a source release or complete
independent reproduction bundle.

Primary CPU claims compare original Brian2 and Brian2-Atlas on the same host,
same eight-core allocation, same one-second biological duration, same 0.1 ms
timestep, same model and same monitoring scope. CUDA and Metal use separately
declared float32 contracts. MPI data remain separate from shared-memory CPU
data. Paper timings from other hardware are not used as speedup denominators.
