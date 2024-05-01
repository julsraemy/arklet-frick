# Arklet-Frick: a basic ARK resolver

Arklet-Frick is a fork of the Internet Archive Arklet (https://github.com/internetarchive/arklet/) project with additional features, improved security, and bugfixes. 

Arklet-Frick is a Python Django application for minting, binding, and resolving ARKS.

It is intended to follow best practices set out by https://arks.org/ (https://arks.org/).

## Feature Comparison

|                              |              |        |
| :--------------------------- | :----------- | :----- |
|                              | Arklet-Frick | Arklet |
| ARK Resolution               | ✓            | ✓      |
| ARK minting and editing      | ✓            | ✓      |
| Bulk minting and editing     | ✓            |        |
| Suffix passthrough           | ✓            |        |
| Separate minter and resolver | ✓            |        |
| API access key hashing       | ✓            |        |
| Shoulder rules               | ✓            |        |
| Extensive metadata           | ✓            |        |
| ?info and ?json endpoints    | ✓            |        |

## Overview

The Arklet-Frick service consists of four major components:

- A SQL database that stores ARK identifiers and their metadata.
- A “resolver”: A Python Django web application that allows users to input URLS containing ARK identifiers and be redirected to the associated URL.
- A “minter”: a Python Django web application that allows approved users to mint and edit ARKs through a web API and administrators to manage access tokens and shoulders through a web interface.
- A Python command line tool that allows approved users to manipulate ARKs using a command line interface.

The repository comes pre-configured with Docker development and production settings to manage the application. 

## Usage

### ARK Resolution

A resolver running at https://resolver.org/ (https://resolver.org/) will resolve ARKs that follow the format laid out in the ARK Alliance specification.

The ARK must consist of four parts: the “ark” header, a namespace, a shoulder, and an identifier.

A standard ARK might look like this: ark:/12345/ab1fgh234mn

GET https://resolver.org/ark:/12345/ab1fgh234mn (https://resolver.org/ark:/12345/ab1fgh234mn) redirects to the value stored in the URL field of the ARK metadata. If the ARK exists but the URL field is empty, a 404 code is returned. If no matching ARK is found, Arklet-Frick will forward the request to https://n2t.net/ (https://n2t.net/).

GET https://resolver.org/ark:/12345/ab1fgh234mn?info (https://resolver.org/ark:/12345/ab1fgh234mn?info) and GET https://resolver.org/ark:/12345/ab1fgh234mn?ijson (https://resolver.org/ark:/12345/ab1fgh234mn?ijson) return the ARK metadata. The ?info extension returns a human-readable HTML representation, while the ?json extension returns pure JSON. If the ARK is not found in the database, a 404 error is returned. 

GET https://resolver.org/ark:/12345/ab1fgh234mn/any_suffix (https://resolver.org/ark:/12345/ab1fgh234mn/any_suffix) or GET https://resolver.org/ark:/12345/ab1fgh234mn?any_suffix (https://resolver.org/ark:/12345/ab1fgh234mn?any_suffix) will redirect to the value of the URL and “pass-through” the value of the suffix to the destination URL. For instance, if ark:/12345/ab1fgh234mn redirects to English Wikipedia, https://resolver.org/ark:/12345/ab1fgh234mn/Henry_Clay_Frick (https://resolver.org/ark:/12345/ab1fgh234mn/Henry_Clay_Frick) will redirect to  https://en.wikipedia.org/wiki/Henry_Clay_Frick (https://en.wikipedia.org/wiki/Henry_Clay_Frick).

### ARK minting and editing

ARK endpoints require an Authorization header with a valid API key. API keys can be provisioned by the administrator, using the administrator’s interface. API keys are tied to specific NAANs. The minter supports two endpoints and can function as a resolver for GET commands. The minter also supports bulk versions of all three functions.

POST /mint mints the ARK described by the JSON input. Request parameters:

- Naan
- Shoulder
- URL (optional)
- Title (optional)
- Type (optional)
- Commitment (optional)
- Identifier (optional)
- Format (optional)
- Relation (optional)
- Source (optional)

The /mint endpoint returns a JSON response with the minted ARK identifier. 

Both the NAAN and shoulder supplied to the /mint command must already exist when the command is called. Minting requests with an invalid NAAN or shoulder will raise an error.

PUT /update updates an ARK with the metadata supplied in the JSON input. Request parameters:

- Ark
- URL (optional)
- Title (optional)
- Type (optional)
- Commitment (optional)
- Identifier (optional)
- Format (optional)
- Relation (optional)
- Source (optional)

The /update endpoint returns a 200 response if the update is successful.

The three bulk endpoints are /bulk_mint, /bulk_update, and /bulk_query. A maximum of 100 records can be manipulated at once.

POST /bulk_mint mints up to 100 records at once. The JSON input must consist of a “naan” key, where the value is a valid NAAN that all the minted ARKs will be assigned to, and a “data” key, where the value is an array of objects containing ARK metadata. Each object must include a valid shoulder. The other valid fields are the standard ARK metadata fields and they are all optional. 

POST /bulk_mint updates up to 100 records at once. The JSON input should consist of a “data” key, where the value is an array of objects containing ARK metadata. Each object must include a valid ARK that already exists. The other valid fields are the standard ARK metadata fields and they are all optional. 

POST /bulk_query returns the metadata for up to 100 queried ARKs at once.  The JSON input should consist of a “data” key, where the value is an array of objects with one “ark” key. 

### Command-line Tool

The /ui subdirectory contains a command-line tool for interacting with the API. Valid commands are query, mint, update, query_csv, mint_csv, and update_csv. The single-use endpoints require either a valid ARK or a valid NAAN/shoulder combination, followed by optional arguments for each valid metadata field: url, title, type, commitment, identifier, format, relation, source.

python arklet_api.py query --ark ark:/13960/t5n960f7n

python arklet_api.py mint --naan 13960 --shoulder /a0 --url https://wikipedia.org/

python arklet_api.py update --ark ark:/13960/t5n960f7n --title new_title

The bulk endpoints are manipulated via CSV when using the command line tool. Each bulk command is limited to 100 values.

python arklet_api.py query_csv --csv /path/to/arks.csv

This command queries a batch of ARKs described in a CSV document. The given CSV document must contain a column of ARK identifiers named “ark.” This command returns a JSON array describing metadata for each ARK record.

python arklet_api.py update_csv --csv /path/to/arks.csv

This command updates a batch of ARKs described in a CSV document. The given CSV document must contain a column of ARK identifiers named “ark.” Additional columns (eg, url, title, etc.) are used to update each field in the ARK record. 

python arklet_api.py mint_csv --naan 13960 --csv /path/to/arks.csv

Mint a batch of ARKs described in a CSV document under the given NAAN. The given CSV document must contain a column of extant shoulders named “shoulder.” Additional optional columns (eg, url, title, etc.) are bound to each ARK record. 

### Administrator’s interface

Administrators can access the /admin interface of the minter to manage API access keys, shoulders, NAANs, and other administrators. Administrator access is protected using standard username and password authentication. The first administrator account must be set up when initially deploying the application.

_A note on ARK metadata_

The Arklet-Frick schema is based on the main Dublin Core elements:
- Title (http://purl.org/dc/elements/1.1/title (http://purl.org/dc/elements/1.1/title)): the name of the resource.
- Type: (http://purl.org/dc/elements/1.1/type (http://purl.org/dc/elements/1.1/type)): the intellectual type of the resource, such as collection, image, or text. Values should be controlled and can be specific to the Frick's namespace.
- Format (http://purl.org/dc/elements/1.1/format (http://purl.org/dc/elements/1.1/format)): the file format, medium, or dimensions of a resource at its current access point. An ARK for a bibliographic record in the Alma catalog has the format MARC. Values should be controlled and specific to shoulders.
- Identifier (http://purl.org/dc/elements/1.1/identifier (http://purl.org/dc/elements/1.1/identifier)): the unique identifier of the resource in the system of record.
- Relation (http://purl.org/dc/elements/1.1/relation (http://purl.org/dc/elements/1.1/relation)): typically in the arklet-frick schema, membership in an intellectual collection that spans multiple systems of record. This a linking field.
- Source (http://purl.org/dc/elements/1.1/source (http://purl.org/dc/elements/1.1/source)): a resource, ideally a dereferenceable URI, from which the described resource is derived. Examples include a digital asset derived from a container asset or bibliographic resource. This is a linking field.
No metadata field in the schema is required by the application, but best practice is to provide some value for each field according the shoulder schema. The Name2Thing platform provides structured values for the ARK specification at https://n2t.net/e/n2t_apidoc.html (https://n2t.net/e/n2t_apidoc.html).

## Setup 

Use docker-compose up to automatically launch the Postgres database, the arklet-minter component, and the arklet-resolver component. By default, the minter runs on 127.0.0.1:8001 and the resolver runs on 127.0.0.1:8000.

Configuration for the local environment can be found in the docker/env.local envfile. Note that if you wish to change the ports you also need to update the port forwarding configuration in docker-compose.yml.

### Creating a Superuser
To properly set up the application, create the first user with admin privileges from the command line. 

make dev-cmd

…launches a bash shell in the minter container.

./manage.py createsuperuser

…creates the superuser.

### Admin Panel Setup
To continue the setup, access the Admin panel via a web browser at 127.0.0.1:8000/admin.

Next, create a NAAN and an API key to begin using the web API.

### Production

This repository is pre-configured for production deployment using nginx and guincorn, and assumes the use of a managed database instance and general webserver computing instance. 

To use this repository, fill in the relevant database credentials and a secure Django secret key in env.prod.example and rename the file to env.prod. 

To launch the minter, resolver, and nginx server run docker-compose -f docker-compose.nginx.yml --profile nginx up, or simply make prod. By default, the resolver runs on port 80 and the minter runs on port 8080. To change the port that the minter is accessed on, alter the port numbers in both docker-compose.nginx.yml as well as nginx.conf. 

## License

This code is licensed with the MIT open-source license.
