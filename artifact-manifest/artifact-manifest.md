# OCI Artifact Manifest

The OCI artifact manifest provides a means to define a wide range of artifacts, including a chain of dependencies of related artifacts. It provides a means to define multiple collections of types, including blobs, dependent artifacts and referenced artifacts.

These collections provide the information required for:

- validating an artifact
- registry management and visualizations
- deletion management, including reference counting and garbage collection
- indexing for artifact discover, along with it's related content
- copying within & across registries

## Design Points

OCI Artifact Manifest has the following design points:

### Artifacts Move Within & Between Registries

Artifact will move from dev, through test, to staging. They'll continue movement to a distribution point or deployment. Throughout the process an artifact may move to other repositories within a registry or across different registries.

Content movement within a registry:

```bash
registry.acme-rockets.io\
  dev\
    web-image:v1
    web-image:v2
    web-image:v3
    web-deploy:v1
    web-deploy:v2
    web-deploy:v3
  staging\
    web-image:v2
    web-image:v3
    web-deploy:v2
    web-deploy:v3
  prod\
    web-image:v2
    web-deploy:v2
```

Content movement across registries:

```bash
dev-registry.acme-rockets.io\
  web-image:v1
  web-image:v2
  web-image:v3
  web-deploy:v1
  web-deploy:v2
  web-deploy:v3
```

```bash
prod-registry.acme-rockets.io\
  web-image:v2
  web-deploy:v2
```

Content published for public consumption

```bash
products.wabbit-networks.io\
  net-monitor:v1
  net-monitor:v2
  net-monitor-charts:v1
```

To support artifact movement, the content that constitutes an artifact to individually represent itself must be maintained within the same registry. For example, the layers of an container image, or the signature blob of a Notary v2 signature artifact.

### Deferred Resolution

To support artifact movement to various registry and namespace structures, the registry and path must not be embedded within the artifact definition. Client CLIs and configurations will provide default locations and mappings for where to find the referenced content.

Artifacts that reference other artifacts must include an OCI Artifact Descriptor which includes the `manifest type`, `digest`, `size` and `repo:tag` of the artifact, however it will defer resolution of the reference to client tools that MAY reconstitute the references from multiple repositories and/or registries.

Clients MAY choose to bind to specific digests, assuring they are testing and using the exact graph initially specified. Or, clients may choose to float to a newer version of a tag, benefiting from patches. In all cases, a Notary v2 signature may be used to assure the artifacts are true to their initial ownership and authors.

**Setting the default registry:**

```bash
oci-reg default-registry registry.acme-rockets.io
```

**Setting repository mappings for where to push and pull unqualified artifacts:**

`oci-reg.config`

```json
{
  "root-namespace": "dev"
}
```

**Setting repository mappings for specific artifacts:**

`oci-reg.config`

```json
{
  "default-registry": "registry.acme-rockets.io",
  "root-namespace": "prod",
  "repo-mappings": [
    {
      "repo": "wordpress-chart",
      "path": "/charts"
    },
    {
      "repo": "wordpress-cnab",
      "path": "/cnabs"
    }
  ]
}
```

## Supported Artifact Types

Artifact manifest is intended to support the following artifact types:

### OCI Image

An OCI Image is based on the oci-image-manifest. It's shown as a comparison to the examples below.

![OCI Image](media/wordpress-image-layers.svg)

### Notary v2 Signature Persistance

A Notary v2 signature would persist as a manifest with a config object and a signature, persisted as a blob. However, the signature has no value unto itself. A signature is applied to a given artifact. It's said to be dependent upon another artifact to be complete.

![notary v2 signature](media/notaryv2-signature.svg)

The Notary v2 signature would reference an artifact, such as the `wordpress:v5` image above. Notice the directionality of the references. One or more signatures may be added to a registry after the image was persisted. While an image knows of it's layers, and a Notary v2 signature knows of its config and blob, the Notary v2 signature declares a dependency to the artifact it's signing. The visualization indicates the references through solid lines as these reference types are said to be hard references. Just as the layers of an OCI Image are deleted (*ref-counted -1*), the blobs of a signature are deleted (*ref-counted -1*) when the signature is deleted. Likewise, when an artifact is deleted, the signature would be deleted (*ref-counted -1*) as the signatures have no value without the artifact they are signing.

![wordpress image with layers](media/wordpress-image-layers-sig.svg)

### Helm Charts & CNAB

A Helm chart can represent the images it references within the chart. These references are loose references as they may be persisted in different registries, or may change as a values file is updated. However, the chart may also be persisted together as a collection of artifacts in a registry. The lines are dotted to represent the loose reference. Deleting the `wordpress-chart:v5` may, or may not delete the images as the images have value unto themselves.

![Wordpress Helm Chart](media/wordpress-helm-chart.svg)

A CNAB may also be persisted with configuration information, along with a reference to its invocation image. The reference to the `helm-cli:v3` is solid line/hard reference. This allows the helm-cli to be deleted *(ref-count -1)* when the parent cnab is deleted. As the CNAB references a Helm chart, the `wordpress-chart:v5` is also represented as a loose reference as the helm chart and referenced images have value unto themselves.

![Wordpress CNAB](media/wordpress-cnab.svg)

## Supported Scenarios

The main scenarios include:

1. Discovery of content within a registry for content listing through CLI and visualizations.
1. Copying within and across registries.
1. Deletion management, providing information to de-dupe content with reference counting.
1. Support enhancing information related to existing content. Such as adding a Notary v2 signature or SBoM.
1. Validation, with required and optional references.

### Content Discovery

Registries today support a flat list of content within designated repositories. A container image, multi-arch container image, Helm Chart, CNAB, Singularity, WASM and other OCI Artifact types can be listed based on their `manifest.config.mediaType`

![flat listing of OCI artifacts](media/repo-listing-flat.svg)

In the above example, all the artifacts are displayed without relation to each other. The layers of the `:v5` are also displayed as an example of data that is already hidden.

![flat listing of OCI artifacts](media/repo-listing-attributed.svg)

In the above example, the Notary v2 signature, an SBoM and collection of attributes are displayed as directly associated with their primary artifact.

![flat listing of OCI artifacts](media/repo-listing-attributed-expanded.svg)

In the above case, the graph of references can be expanded showing the references across repositories. This visualization demonstrates the hierarchy known by the registry, based on the artifact-manifest. While an artifact icon is displayed, based on the `manifest.config.mediaType`, the registry need not know any of the artifact specific details to track this data. Registries would not need to parse the Helm Chart, the CNAB. Each artifact author will lift data elements for the objects they wish to reference in a registry.

### Copy Container Images

![mysql image copy](media/mysql-copy.svg)

As a reference, copying a container from a public registry to a private registry would involve `docker pull`, `docker tag` and `docker push`

```bash
docker pull mysql:8
docker tag mysql:8 registry.acme-rockets.io/base-artifacts/mysql:8
docker push registry.acme-rockets.io/base-artifacts/mysql:8
```

The above commands account for the image manifest and the associated layers. Note the directionality of the manifest `-->` layers references. A manifest declares the layers that must be accounted for before a manifest may be considered valid within a registry. In addition, most registries implement layer de-duping and reference counting to avoid maintaining multiple copies of the same layers.

```json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7097,
    "digest": "sha256:c8562eaf9d81c779cbfc318d6e01b8e6f86907f1d41233268a2ed83b2f34e748"
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 27108069,
      "digest": "sha256:a076a628af6f7dcabc536bee373c0d9b48d9f0516788e64080c4e841746e6ce6"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 1741,
      "digest": "sha256:f6c208f3f991dcbc417fed8efde391f887c0551d77ed0c1a125fd28f4841e1cc"
    }
  ]
}
```

### Container Image, with Signatures

![mysql image copy](./media/mysql-with-sigs-copy.svg)

In this scenario, Notary v2 signatures have been added. Note the directionality of the Notary v2 signatures references. The `mysql:8` image has no reference to the signatures. The signatures may be added to existing artifacts. The signatures declare a reference `-->` to the `mysql:8` image. This assure the `mysql:8` image tag and digest need not change to support signatures added throughout the secured supply chain.

From a user experience perspective, copying a container from a public registry to a private registry would likely be expected to copy the signatures alongside the artifact they've signed.

The `oci.artifact.manifest` supports the Notary v2 requirements, including:

- support for additive signatures, assuring the target manifest digest and tag do not change
- support for multiple signatures. In the above scenario, `mysql` signed the original image. As `mysql` was copied to Docker Hub, an additional `docker community signature` was added, providing certified content attestation. Once the image copy to ACME Rockets is completed, an additional `acme-rockets signature` is added providing assurance the `mysql:8` image was security scanned and verified it's applicable to the ACME Rockets environment.

From a user experience, the signature artifacts have no unique value beyond the artifact they represent, therefore they would be persisted to a registry in a form by which they are known to copy with the artifact, and be deleted when the artifact they are associated with is deleted.

To support hard references, an additional dependencies collection is added to a new `application/vnd.oci.artifact.manifest.v1+json` schema. The `dependencies` collection declares the artifact the signature is dependent upon. Similar to pypi packages, the validation of dependencies are deferred. If the dependent artifacts are found, they are valid. Depending on the validation options, a signature may exist without its dependent artifact.

**A Notary v2 signature of the `mysql:8` image example:**

```json
{
  "mediaType": "application/vnd.oci.artifact.manifest.v1+json",
  "artifactType": "application/vnd.cncf.notary.v2",
  "config": {
    "mediaType": "application/vnd.cncf.notary.config.v2",
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7",
    "size": 102
  },
  "blobs": [
    {
      "mediaType": "application/vnd.cncf.notary.v2.json",
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0",
      "size": 32654,
      "reference": "registry.wabbitnetworks.io"
    }
  ],
  "dependencies": [
    {
      "artifact": "mysql:8",
      "artifactType": "application/vnd.oci.image.manifest.v1.config.json",
      "mediaType": "application/vnd.oci.image.manifest.v1.config.json",
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b",
      "size": 16724,
    }
  ]
}
```

#### OCI-Registry CLI

To copy the above image and the associated signatures, a new `oci-reg` cli is proposed. The oci-reg cli is an independent tool that demonstrates the value of these collections, providing a unified standard means for working within and across different OCI compliant registry implementations.

The following command would copy the `mysql:8` image from docker hub to the acme-rockets registry. The CLI could be run within the source or target cloud. 

```bash
oci-reg copy \
  --source hub.docker.io/mysql:8 \
  --target registry.acme-rockets.io/base-artifacts/mysql:8
```

The `oci-reg copy` command would:

- assure the manifest and layer/blob digests remain the same
- copy any artifacts that are dependent on the source artifact-manifest, persisting them in the target registry.

### Reference Artifacts

There are a set of artifact types that declare references to other artifacts that may, or may not be stored in the same registry. The reference is important to note, indicating copying to be capable between environments, as well as generalized validation scenarios.

#### Helm Reference

![mysql image copy](./media/wordpress-helm-chart-copy.svg)

In the above scenario, a helm chart is copied from a public registry to the ACME Rockets registry.  The `wordpress-chart:v5` is represented as an `application/vnd.oci.artifact.manifest.v1+json`. The `wordpress-chart:v5` helm chart references the `wordpress:v5` image and the `mysql:8` image. All three artifacts have signatures attesting to their authenticity.

As the copy is initiated, the `oci.artifact.manifest` of the `wordpress-chart:v5` is evaluated. As the chart references the same version (digest) of the `mysql:8` image already in theACME Rockets registry, the copy skips duplicating the content and moves to copying the `wordpress:v5` image, the `wordpress-chart:v5` and their associated signatures.

In this case, the images referenced in the chart existed in the source registry. However, this may not be the case in all scenarios. A helm chart may be acquired from a different location, referencing images from docker hub. This deferred validation is left to client tools, enabling common package management resolution scenarios.

To support the loose references between artifacts, a `references` collection is added to the `oci.artifact.manifest`:

**A `wordpress-chart:v5` Helm Chart example:**

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.artifact.manifest.v1+json",
  "artifactType": "application/vnd.cncf.helm.v3",
  "config": {
    "mediaType": "application/vnd.cncf.helm.config.v1+json",
    "size": 0,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
  },
  "blobs": [
    {
      "mediaType": "application/vnd.cncf.helm.chart.v1.tar",
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0",
      "size": 32654
    },
    {
      "mediaType": "application/vnd.cncf.helm.values.v1.yaml",
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b",
      "size": 16724
    }
  ],
  "references": [
    {
      "artifact": "wordpress:5.7",
      "artifactType": "application/vnd.oci.image.manifest.v1.config.json",
      "mediaType": "application/vnd.oci.image.manifest.v1.config.json",
      "digest": "sha256:5c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c82",
      "size": 1510
    },
    {
      "artifact": "mysql:8",
      "artifactType": "application/vnd.oci.image.manifest.v1.config.json",
      "mediaType": "application/vnd.oci.image.manifest.v1.config.json",
      "digest": "sha256:8c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c31",
      "size": 1578
    }
  ]
}
```

#### CNAB Reference

A CNAB is yet another reference artifact. While the current CNAB spec incorporates the helm-cli ahd helm chart within an invocation image, the `artifact.manifest` provides more natural package management experiences where the references can be resolved based on the users intent, while leveraging the capabilities of an OCI compliant registry to store all OCI Artifact types.

![mysql image copy](./media/wordpress-cnab-copy.svg)

Similar to the Helm example, a CNAB is copied from a public registry to the ACME Rockets registry. The `wordpress-cnab:v5` CNAB declares references `-->` to an invocation image that includes the `helm-cli:v3`. This invocation image provides an environment to run `helm install` of the referenced `wordpress-chart:v5`. The CNAB includes an additional reference `-->` to `wordpress-chart:v5`. The chart includes references `-->` to the `wordpress:v5` and `mysql:8` images. Lastly, each artifact has a Notary v2 signature that points to `<--` the artifact they are signing.

As the `oci-reg copy` command is executed, the graph of references are expanded. As the copy proceeds, only those artifacts that don't already exist in the target registry are required to be copied. The references may be hard bound to the digest, or loosely bound to the `artifact:tag` enabling more recently patched versions of a given `artifact:tag`. The CNAB and Helm `artifact.manifest` may declare how strict they wish to couple their references to **stable tags** or **unique digests**

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.artifact.manifest.v1+json",
  "artifactType": "application/vnd.cncf.cnab.v1",
  "config": {
    "mediaType": "application/vnd.cncf.cnab.config.v1+json",
    "size": 0,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
  },
  "blobs": [
    {
      "mediaType": "application/vnd.cncf.cnab.v1.tar",
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0",
      "size": 32654
    },
    {
      "mediaType": "application/vnd.cncf.cnab.params.v1.json",
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b",
      "size": 16724
    }
  ],
  "references": [
    {
      "artifact": "wordpress-chart:v5",
      "artifactType": "application/vnd.cncf.helm.v1.config.json",
      "mediaType": "application/vnd.oci.artifact.manifest.v1+json",
      "digest": "sha256:5c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c82",
      "size": 1510
    },
    {
      "artifact": "helm-cli:3",
      "artifactType": "application/vnd.oci.image.manifest.v1.config.json",
      "mediaType": "application/vnd.oci.image.manifest.v1.config.json",
      "digest": "sha256:8c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c31",
      "size": 1578
    }
  ]
}
```

#### Nydus Image Reference
![nydus image copy](./media/nydus_image.png)

### Deletion

Distribution-spec APIs will provide standard delete operations, including options for deleting referenced artifacts, or blocking a delete as the artifact is referenced by other artifacts. The `oci.artifact.manifest` collection will provide the information, as defined by the artifact author` for how an artifact should be handled for delete operations. The registry, nor the `oci-reg` cli would need to know about specific artifact implementations.

The deletion scenarios include:

- Which references should be deleted (ref counted)
- Which artifacts may be blocked from deletion as another artifact depends upon it.

- Examples:
  - deleting the wordpress helm chart deletes the config, chart and values blobs.
  - deleting the wordpress helm chart will reduce the ref-count of the images it may reference.
  - deleting the mysql image should warn if referenced by any helm charts. However, based on the parameters of the cli, it may delete images referenced by helm charts within the registry as these references are considered loose references and may be resolved externally to the registry.
  - deleting the wordpress helm chart, wordpress image, or mysql image would delete the associated signatures as the signatures have no value unto themselves. The `oci-reg` cli may accomplish these delete steps without any knowledge of Notary v2 spec as the `oci-reg` delete cli will follow the rules of the `oci.artifact.manifest` collections.

## OCI Artifact Manifest Content Descriptor

The OCI Artifact Manifest content descriptor is enhanced with an additional `artifact` property enabling loose or hard references between artifact types.

- **`mediaType`** *string*

  This property identifies the OCI schema of the manifest being referenced.
  An OCI v_ compliant registry must support at least the following schemas:

  - `"application/vnd.oci.image.manifest.v1+json"`
  - `"application/vnd.oci.image.index.v1+json"`
  - `"application/vnd.oci.artifact.manifest.v1+json"`

- **`digest`** *string*

  This REQUIRED property is the digest of the targeted content, conforming to the requirements outlined in Digests. Retrieved content SHOULD be verified against this digest when consumed via untrusted sources.

- **`size`** *int64*

    This REQUIRED property specifies the size, in bytes, of the raw content. This property exists so that a client will have an expected size for the content before processing. If the length of the retrieved content does not match the specified length, the content SHOULD NOT be trusted.  

- **`artifact-type`** *string*

  This property defines the OCI Artifact Type. In previous versions of the OCI Artifact spec, this was defined with `manifest.config.mediaType`. This property is lifted and formalized in the `oci.artifact.manifest` enabling filtering of references to those of a given `artifact-type`, such as returning a list of Notary v2 signatures.

- **`artifact`** *string*

  This property provides a reference to the `artifact:tag`, enabling floating to a newer/patched version of a tag. An OCI Artifact Manifest content descriptor must include the digest to assure a client may validate the initially referenced artifact, however this is a client choice for binding to a specific digest, or newer version based on the `artifact:tag` named reference.

## OCI Artifact Collections

OCI Artifact Manifests provide the following types of references:

- **Blobs:** Content that represents the artifact. These are analogues to layers from the OCI Image manifest and Config objects. Layers are renamed blobs as they represent a generic collection of content, as opposed to an ordered layered collection as defined by OCI Image Manifest. An artifact may treat them as ordered, but it is not required.
- **Dependencies** on other artifacts that enhance the content, such as a Notary v2 signature or an SBoM. These dependencies are *unknown* by the original artifact as they are added at a later time. A registry would need to index these references as registry apis would request all content related to the source artifact.
- **References** to other artifacts, used to complete a scenario, but may not be stored within the same repository or registry. These references are defined by the source artifact and known at the time of upload to a registry, such as a Helm chart that references other images. These references are included in the manifest and computed in the digest of the manifest.

### Blobs Collection

All blobs are considered to be hard dependencies that must be resolvable within a registry. An artifact is considered invalid if the manifest blobs are not resolvable. Registries MAY implement de-duping, using ref-counting to assure at least one copy of the blob is resolvable for any given `oci.artifact.manifest`. OCI Artifact blobs are generalizations of the OCI Image Spec layers definition.

### Dependencies Collection

Alternate names:

- `dependent-upon`
- `parents`
- `enhances`

The dependencies collection is an optional collection of references to other artifacts. The artifact is said to enhance the dependent artifacts by adding additional content. The content may be added after the initial content was created or pushed to a registry. By supporting additional content, the referenced artifact can be enhanced without having to change the referenced artifacts manifest, digest or tag.

Examples include:

- Notary v2 signatures
- SBoM documents
- Artifact Meta-data

Dependencies are collections of OCI Artifact Content Descriptors.

### References Collection

The references collection is an optional collection of loose references to additional artifacts that complete a given scenario. The references may be stored in other repositories and/or other registries. The references collection allows an artifact to define a graph of content, used by it's client tools. While providing generic references to a registry that maintains the references without having to validate them for artifact submission or removal from a registry.

References are made to specific content digests and tags, however it is up to the client to determine how to resolve the reference. The client MAY bind directly to the content digest, or bind to the tag, allowing newer patched versions of the tag. In all cases, a Notary v2 signature MAY be used to assure the content derives from the original authority.

Examples include:

- A helm chart referencing container images
- A CNAB referencing Helm charts or other artifacts the CNAB may need to complete it's operation
- A WASM that may reference other packages that may be stored in a registry.

References are collections of OCI Artifact Content Descriptors.

## Annotations

OCI Artifact Manifest includes several annotations that have been generalized from the image-spec annotations.

- **`annotations`** *string-string map*

    This OPTIONAL property contains arbitrary metadata for the image manifest.
    This OPTIONAL property MUST use the [annotation rules](annotations.md#rules).

    See [Pre-Defined Annotation Keys](annotations.md#pre-defined-annotation-keys).

### Pre-Defined Annotation Keys

This specification defines the following annotation keys, intended for but not limited to  Artifact Manifest authors:

- **org.opencontainers.artifact.created** date and time on which the artifact was built (string, date-time as defined by [RFC 3339](https://tools.ietf.org/html/rfc3339#section-5.6)).
- **org.opencontainers.artifact.authors** contact details of the people or organization responsible for the artifact (freeform string)
- **org.opencontainers.artifact.url** URL to find more information on the artifact (string)
- **org.opencontainers.artifact.documentation** URL to get documentation on the artifact (string)
- **org.opencontainers.artifact.source** URL to get source code for building the artifact (string)
- **org.opencontainers.artifact.version** version of the packaged software
  - The version MAY match a label or tag in the source code repository
  - version MAY be [Semantic versioning-compatible](http://semver.org/)
- **org.opencontainers.artifact.revision** Source control revision identifier for the packaged software.
- **org.opencontainers.artifact.vendor** Name of the distributing entity, organization or individual.
- **org.opencontainers.artifact.title** Human-readable title of the artifact (string)
- **org.opencontainers.artifact.description** Human-readable description of the software packaged in the artifact (string)

## Open Questions

Should the references collection support additional types, like loose urls
