### Introduction

Code packages are copied over the network, packages are decompressed by the CPU, the decompressed files are written to disk, and application code and state must be loaded into cold memory.  All of these costs directly correlate with the size of the deployment bundle.

The platform requires users to write services on a series of standard libraries, so that they can be preloaded quickly. At the same time, the platform also requires developers to follow a fixed pattern for faster deployment. However, this unreasonable requirement greatly limits development, and it is basically difficult to build projects that cannot use popular libraries at present. 

To solve this problem, it's natural to try to decouple the decencies with containers.  One way to do this is rewrite old packages, splitting these packages into smaller bundles, so the functions can only load the needed part. However, it's a simple but no-way method, we can not just rewrite all the packages.

We propose another level into serverless system, which hold all packages needed, and can be shared by all containers without costs.

However, this thought is no easy task, In uncompressed form, the cumulative size of the packages is 1.3 TB. That's why we need Pipsqueak, a package caching mechanism.

### Mechanism

To cold use a package, the following steps are needed: Download, Install ( decompressed, written to disk and compile phase ), Import ( there will be a CPU cost to generating Python bytecode  )

We would like to have its necessary packages already downloaded, installed, and imported, since cache all packages is not possible, so a good mechanism to decide which should be cached is important.

At the same time, the **security** assumptions is also be considered, so we do the following things:

1. Use a sandbox to cache these packages
2. Maintain a handler isolated with the packages.
3. No action on active usage 

#### Interpreter Forking  

Caching initialized modules in sleeping Python interpreters. In fact, after invoke a cache, we can introduce more packages , which can be considered as a child node of the current cache.
So how to choose cache node and delete from the pools is also a interesting question.



