 # GCViT
![GCViT](assets/readme_images/logo.png?raw=true)

## Table of Contents
+ [About](#about) 
+ [Setup](#setup)
    + [General](#general-setup)
    + [Docker](#docker-setup)
    + [Go + Node](#go-+-node-setup) 
+ [API](#api)

## About

GCViT is a tool for whole genome visualization of resequencing or SNP array data, which reads data in GFF and VCF format and allows a user to compare two or more accessions to visually identify regions of similarity and difference across the reference genome. Access to data sets can be controlled through authentication.

GCViT is built on top of [CViTjs](https://github.com/LegumeFederation/cvitjs), a Javascript application for viewing genomic features at the whole-genome scale. GCViT is implemented in Go. A Docker image is available. GCViT exposes an API, and can be installed as a server, with no UI.

![Williams Pedigree As Haplotype Blocks ](assets/readme_images/Williams_Pedigree2.png?raw=true)
Figure 1. An example of haplotype comparisons of 6 soybean accessions.

[Explore Soybean SNP data in GCViT](https://soybase.org/gcvit/)

## Setup
While GCViT is intended as a tool for publicly accessible data, it may also be run locally as a stand alone tool. In either case, there are two main approaches to running GCViT, in a Docker container, or using the built-in Go server. The configuration of the backend service and the UI stays mostly the same in either case.
A stand-alone Electron desktop app is in the process of being developed. 

Instructions for the UI are provided in the application itself.
 
### General Setup

The steps for setting up a GCViT instance consists of downloading and installing the application, configuring the server, and data preparation. The GCViT repository includes example data from soybean consisting of these files: SNP data is in `assets/SoySNP50k_TestFile_named.vcf,` the backbone chromosomes are defined in `ui/cvit_assets/data/soySnp/gm_backbone.gff,` and the CViTjs image is configured with `ui/cvit_assets/data/soySnp/soySnp.conf.` 

#### Configuring the Service
No matter which method you intend to run GCViT, configuration of the Go backend service is the same. The default configuration file is `config/assetsconfig.yaml` and it has the following format:

```yaml
server:
  port: 8080
  portTLS: 8888
  certFile: config/testcert.cert
  keyFile: config/testcert.key
  apiOnly: False
  source: gcvit
  binSize: 500000

users:
  username : password

snptestLegacy:
  location: assets/SoySNP50k_TestFile_named.vcf.gz 
  name: soySNP 50k subset [named]
  format: vcf
  restricted:
    - username
```

The server stanza is optional, and supports the following options:

| Option | Default | Use |
| ----- | ----- | ----- |
| port | 8080 | Changes the port GCViT listens on for HTTPS traffic. Defaults to 8080 only if no portTLS is provided. Otherwise ignores HTTP traffic. |
| portTLS | - | Changes the port GCViT listens for HTTPS traffic. No default provided as you need to set your own key/cert. |
| certFile | - | Cert file for HTTPS. config/testcert.cert is only for testing purposes and not a default. |
| keyFile | - | Key file for HTTPS. config/testcert.key is only for testing purposes and not a default. |
| apiOnly | False | If True, only serves the api routes, ignoring the GCViT frontend |
| source | gcvit | Value for Column 2 of generated gff files from /api/generateGFF |
| binSize | 500000 | Default number of bases used for bins |

The users stanza is also optional. Use this configuration option to set one-or-more users to password protect datasets.
Without proper credentials, users will never be presented with restricted datasets when using the gcvit ui.
The format is one-or more `<username> : <password>` pairs. Note this only uses BasicAuth headers, and isn't intended to 
be very secure. Future updates may include better practices if demand is present.

Finally you may have one or more data tracks, that have the following required fields:

```yaml
key: #internal key for API requests
  location: relative to root of server directory
  name: display name for dropdowns
  format: vcf (only option for now, automatically checks if gzipped)
  restricted: [optional] whitelist of users that may access this dataset, if not present, data may be accessed by anyone
    - username: username that can access this data
    - username2: another user that can access this datta
```

While it is recommended, the data file given for 'location' does not have to be in the `assets` folder to be read by GCViT.

### Configuring the UI

In addition to the vcf file, a gff "backbone" file will need to be provided for the UI component to display the results. The format for this file can be viewed at `ui/cvit_assets/data/soySnp/gm_backbone.gff`. CViTjs configuration files will need to be edited as well. See the example files `ui/cvit_assets/cvit.conf` and `ui/cvit_assets/soySnp/soySnp.conf`.

Most aspects of the CViTjs display can be customized, including colors, fonts, and the popover box that appears when mousing over a feature. For more information on configuring the CViTjs component of GCViT, please see the documentation [HERE](https://github.com/LegumeFederation/cvitjs/wiki)

Configuration files for the three glyphs used by GCViT *Haplotype Block*, *Heatmap* and *Histogram* are in `ui/src/Components/[HaploConfig.js|HeatConfig.js|HistConfig.js]` respectively.

Other default options (title, bin size, ruler tic interval) can be changed through editing the values in `ui/src/Components/DefaultConfiguration.js`. After changes are made, the docker container will need to be rebuilt, or a manual build will need to be  triggered through node, as described in the following sections. 

**Note:** Configuration settings in `ui/src/Components/DefaultConfiguration.js` override CViTjs equivilent configuration settings, for example, ruler tic interval.

### Docker Setup
For general use, it is probably easiest to get started with GCViT using [Docker](https://www.docker.com/).
Before starting, make sure that docker is properly configured for your system.

The process of building from docker will automatically grab the most recent version of CViTjs during the build process.
To make any changes, including your custom backbone and configuration files, place the files in `ui/cvit_assets`. Any files here
will replace-in-place their equivalent in the default CViT package, this includes custom popover components.

To build through docker:
```
docker build -t gcvit:1.0 . -f Dockerfile
```
This will produce a image with the tag of **gcvit:1.0** that can be used to build a container.
If you want to save time with automated builds and only need the server API component, the build-arg:
```
--build-arg apionly=false
```
is provided to skip over the building of the UI components.
Similarly, if you wish to build the tool with BasicAuth the build-arg:
```
--build-arg apiauth=true
```

When starting the container, there are two mount points exposed to add configuration and data directories:
`/app/config` and `/app/assets` respectively.

An example of starting an instance of GCViT inside the gcvit directory: 
```
docker run -d \
--name gcvit \
--mount type=bind,source="$(pwd)"/config,target=/app/config \
--mount type=bind,source="$(pwd)"/assets,target=/app/assets \
-p 8080:8080 \
gcvit:1.0
```

To update the data, you should be able to just add it directly to the mounted source, as GCViT checks for updated data when appropriate. 

##### Modifying the Docker container
After building the Docker container, you will see three directories in the `ui/` directory: `build/,` `cvit_assets/,` and `public/.` 

To make changes that will take affect when you build the container, make the changes in `build/.` Mirror cvit's directory/name structure for this and it will replace-before-build.

To make changes without rebuilding the GCViT container but that require rebuilding CViTjs, edit and add files to `public/.`

To make changes without rebuilding GCViT or CViTjs, edit and add files to `build/.`

The best practices are to make CViTjs changes in `cvit_assets/` or `public/.`

### Go + Node Setup
GCViT may also be built and served directly using [Go](https://golang.org/) and and [Node](https://nodejs.org/en/) together.
Before beginning, check that Go is set up and Node is configured to at least the most current LTS version (currently 12.14.0).

#### Building the backend component with Go
The following packages are needed in order to build the service:
```
github.com/awilkey/bio-format-tools-go/gff 
github.com/awilkey/bio-format-tools-go/vcf
github.com/go-ozzo/ozzo-routing
github.com/spf13/viper
github.com/golang/gddo/httputil/header
```
It is recommended to use [dep](https://golang.github.io/dep/) to manage dependencies, as running `dep ensure` will make sure the project has all the right requirements on hand.

Once all the dependencies are grabbed, you can run the test server using the command:

`go run server.go`
from the root directory of the project.

By default this will listen on port 8080, but you can change this in the configuration file.

Once up, you can test that the server is up using wget or curl:

| protocol | request |
| -------- | ------- |
| wget     | wget localhost:8080/api/experiment |
| curl     | curl localhost:8080/api/experiment |

Either protocol should return a JSON object that contains a list of all the configured VCF files.

If you wish to compile the server, Go has a robust set of tools for building and cross-compiling binaries.
In the most simple form:

`go build -o server .`

Will builds a binary that has statically linked libraries, making it portable.

If running on a server without a Go compiler configured, the language has support built-in for cross compiling built in. See [HERE](https://golangcookbook.com/chapters/running/cross-compiling/)
for details.

#### Building the frontend components with Node

This repository contains a pre-built version of both the UI component and CViTjs, so this section is mostly optional.

If you wish to add configuration data for CViT without re-building the tool, you may place the files directly in `ui/build/cvitjs`.

If you want to use a custom build of CViTjs, grab and build [CViTjs](https://github.com/LegumeFederation/cvitjs/tree/preact/buildalt). Place the resulting
files from the build directory directly into `ui/build/cvitjs/build`. 

Any changes to CViT that you want to keep when re-building the UI component will need to be placed in `ui/public/cvitjs`

To rebuild the UI component of GCViT:
```
npm install
npm run build
```
or 
```
npm install
npm run buildauth
```
if you wish to enable the basic authentication login prompt.

This will create a webpacked version of the GCViT UI. Most common reasons to rebuild is updating the Help documentation and
updating the CSS.


## API:

The following API is served by the GCViT service component:

| Path | Verb | Returns |
| ---- | ---- | ---- |
| /api/experiment| GET | JSON representation of all experiments in assetconfig.yaml |
| /api/experiment/{experiment} | GET | JSON representation of all PIs in VCF header |
| /api/generateGFF | POST | returns gff. Expected parameters of Ref={experiment:PI}&Variant={sameexperiment:PI}, with any number of variants |
| | | |
| / | GET | tool UI - Only if apiOnly is **False** |
|/login | GET | Attempts to authenticate a username and password. Returns statis 200 if OK, 401 if not. | 
