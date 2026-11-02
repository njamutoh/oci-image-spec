# oci-image-spec
Hands-on OCI Image Spec lab: manually package a rootfs into layers, blobs, manifests, and an OCI Image Layout, then unpack it back into a runtime bundle for runc.


# OCI Image Basics

A small hands-on lab to understand how an OCI image packages a filesystem before that filesystem is handed to a runtime such as `runc`.

## Goal

The goal of this project was to connect the **OCI Image Specification** to the **OCI Runtime Specification**.

In the previous runtime lab, I started with an already-prepared runtime bundle:

```text
rootfs + config.json
        ↓
       runc
        ↓
    container
```

That worked, but it left one important question unanswered:

> **Where does the `rootfs` normally come from?**

That is where the OCI Image Specification comes in.

The image side is responsible for packaging filesystem content and image metadata in a standard format so that higher-level tools can store, transfer, verify, and unpack it before runtime.

At a high level:

```text
OCI Image
   ↓
filesystem layer(s) + image metadata
   ↓
unpack
   ↓
rootfs
   ↓
OCI Runtime Bundle
   ↓
runc
```

---

## What I Built

I reused the Python root filesystem from my OCI Runtime Bundle project and manually turned it into a simple OCI Image Layout.

The process was:

```text
rootfs
  ↓
layer.tar
  ↓
SHA256 digest
  ↓
blob
  ↓
image config
  ↓
manifest
  ↓
index.json
  ↓
OCI Image Layout
```

The resulting layout looked like:

```text
image/
├── oci-layout
├── index.json
└── blobs/
    └── sha256/
        ├── <layer digest>
        ├── <config digest>
        └── <manifest digest>
```

The files inside `blobs/sha256/` are not given friendly names such as `layer.tar` or `manifest.json`.

Instead, OCI stores them using their SHA256 digest.

This is called **content-addressable storage**.

For example:

```text
layer.tar
   ↓
SHA256
   ↓
8f31c9...
   ↓
blobs/sha256/8f31c9...
```

If the contents of the layer change, the digest changes as well.

This allows container tooling to verify content integrity and identify identical blobs reliably.

---

## 1. Filesystem Layer

The existing Python `rootfs` was first archived into a tar file:

```bash
sudo tar -C source-rootfs -cf layer.tar .
```

For this project, the entire root filesystem was represented as a single layer.

Conceptually:

```text
rootfs
  ↓
filesystem layer
```

Real container images often contain multiple layers, but one layer is enough to understand the relationship.

The layer digest and size were then calculated:

```bash
LAYER_DIGEST=$(sha256sum layer.tar | awk '{print $1}')
LAYER_SIZE=$(stat -c%s layer.tar)
```

The layer was stored in the OCI blob store using its digest:

```bash
cp layer.tar image/blobs/sha256/$LAYER_DIGEST
```

---

## 2. Image Config

The next object was the **image config**.

This should not be confused with the OCI Runtime `config.json`.

The image config describes image-level metadata such as:

- operating system
- CPU architecture
- filesystem layer identities
- working directory
- default command

For this image, the default process is:

```text
/usr/bin/python /app/app.py
```

The image config itself was also hashed and stored as a blob.

So at this stage the blob store contained:

```text
blobs/sha256/
├── <filesystem layer>
└── <image config>
```

---

## 3. Manifest

The **manifest** ties the important pieces of the image together.

It does not contain the filesystem itself.

Instead, it contains descriptors pointing to:

```text
manifest
   |
   ├── image config
   |
   └── filesystem layer(s)
```

A descriptor identifies an object using information such as:

```text
mediaType
digest
size
```

So the manifest effectively says:

> This image uses this configuration and these filesystem layers.

The manifest was also hashed and stored in `blobs/sha256/`.

At that point:

```text
blobs/sha256/
├── filesystem layer
├── image config
└── manifest
```

All three objects were identified by their content digests.

---

## 4. `index.json` and `oci-layout`

The OCI Image Layout also contains two small top-level files.

`oci-layout` declares the version of the OCI Image Layout being used:

```json
{
  "imageLayoutVersion": "1.0.0"
}
```

`index.json` acts as the top-level entry point.

The relationship is:

```text
index.json
    ↓
manifest
   /      \
  ↓        ↓
config    layer(s)
```

This is the main object relationship I wanted to understand from this project.

The image is not one single large file.

It is a collection of content-addressed objects linked together using digests.

---

## 5. Following the Image End to End

Once the image layout was built, I manually followed the references.

First:

```text
index.json
```

points to the manifest.

The manifest then points to:

```text
image config
+
filesystem layer
```

The image config describes the image, while the layer contains the filesystem content.

This gives the complete image-side picture:

```text
index.json
     |
     v
manifest
   /     \
  v       v
config   layer
           |
           v
       filesystem
```

---

## 6. Recreating the Rootfs

To prove that the image layer could recreate the filesystem needed by the runtime, I extracted the stored layer into a new directory:

```bash
sudo tar -C extracted-rootfs -xf image/blobs/sha256/$LAYER_DIGEST
```

After extraction, the original Linux filesystem structure was available again:

```text
app/
etc/
usr/
var/
...
```

This demonstrated:

```text
OCI image layer
      ↓
    unpack
      ↓
    rootfs
```

That extracted filesystem could then be placed into another OCI Runtime Bundle.

---

## 7. Connecting Back to `runc`

The extracted root filesystem was combined with a new runtime `config.json`:

```text
runtime-bundle/
├── config.json
└── rootfs/
```

The runtime configuration was generated using:

```bash
runc spec
```

and the process arguments were set to:

```text
/usr/bin/python /app/app.py
```

The container was then run with:

```bash
sudo runc run image-test
```

Expected output:

```text
hello world
```

That completed the full path:

```text
filesystem
    ↓
OCI layer
    ↓
OCI Image Layout
    ↓
unpack
    ↓
rootfs
    +
runtime config.json
    ↓
runc
    ↓
containerized process
```

---

## Key Concepts

- **Layer** — filesystem content or filesystem changes packaged as an image object
- **Digest** — SHA256 hash identifying exact content
- **Blob** — content stored under its digest
- **Image config** — metadata and defaults associated with the image
- **Manifest** — points to the image config and ordered filesystem layers
- **Descriptor** — identifies another OCI object using its media type, digest, and size
- **`index.json`** — top-level entry point pointing to one or more manifests
- **`oci-layout`** — declares the OCI Image Layout version

---

## What I Learned

The OCI Image Specification and OCI Runtime Specification solve different problems.

```text
OCI Image
= how filesystem content and image metadata are packaged

OCI Runtime
= how a process is executed using that filesystem
```

The relationship between them is:

```text
OCI Image
   ↓
unpack filesystem layers
   ↓
rootfs
   +
runtime config.json
   ↓
runc
   ↓
Linux kernel
   ↓
containerized process
```

This made the container stack much clearer to me.

The runtime does not need to know how the root filesystem was originally built or distributed. It only needs a prepared `rootfs` and runtime configuration.

The image specification is what standardizes the packaging that happens before that point.

---

## Next Step

The next step is the **OCI Distribution Specification**.

That moves the question from:

> **What is inside an OCI image?**

to:

> **How are manifests and blobs transferred between a registry and a container host?**
