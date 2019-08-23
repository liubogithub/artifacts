# Artifacts Table of Contents 

The following is the working plan for the [Artifacts](https://github.com/opencontainers/artifacts) TOC.

## Individual Content PRs

As each section will have various forms of feedback, I'll be breaking up the [latest content](https://github.com/AzureCR/distribution-spec/blob/artifact-registry/artifacts.md) into separate PRs. 

This working TOC gives a bigger picture perspective of what's currently known. *("We don't know what we don't know")* - so, help us know what you'd like to see.

## Stable Links

As others provide links to content, moving content between different files and headings makes it more difficult to maintain those links. Getting the TOC (files and headings fixed) will minimize churn to consumers.

If you have other content you'd like to see added, please comment here so we can structure the files and headings appropriately. 

Comments including:
* Needed Content (sections & scenarios)
* File Naming
* Heading Names, Within the Files

## Working TOC

* Readme.md
    * What makes an artifact
    * Artifacts as well known types
    * Support by all registries - that support OCI Distribution

* For Distribution Operators - Supporting Artifacts	(supporting-artifacts.md)
    * Supporting Artifacts in a Registry
    * Well Known Types *([consuming the list of published artifact types](https://github.com/AzureCR/distribution-spec/blob/artifact-registry/artifacts.md#registering-artifact-types))*
    * Displaying the Type, [with Icons and Human Strings](https://github.com/SteveLasker/RegistryArtifactTypes/blob/master/mediaTypes.md#media-types)
    * Validating Artifacts
    * Importing Supported Artifacts to a Registry
    * Discovering Supported Artifacts *(per registry, per repo)*

* For Artifact Authors - Authoring and Publishing New Artifacts (authoring-artifacts.md)
  * Authoring new artifacts
  * Defining mediaTypes 
    * unknown reserved
    * oci-image for docker/containerd 
  * Annotations and config
  * Optional - Config Schema 
  * Publishing new artifacts
* Supported Registries ([implementations.md](https://github.com/SteveLasker/artifacts/commit/6c090374ea2a847a404b24769b20a99bf688971b))

	artifact-types\folder
		artifact-schema.json
	Consuming 
