# ZipStream Service

[![Build Status](https://travis-ci.org/Cadasta/ZipStream.svg?branch=master)](https://travis-ci.org/Cadasta/ZipStream)
[![Docker Stars](https://img.shields.io/docker/stars/cadasta/zipstream.svg)](https://hub.docker.com/r/cadasta/zipstream/)

A microservice to build and stream dynamically zipped bundles of remote files. The service does not store files, rather it stores references to files which it can stream in a zipped package to users. Designed to be backend-swappable, it is capable of working with just about any database and any filestore. It aims to be fast, have a low memory footprint, and to support tens-of-thousands of concurrent connections <sup>[[citation needed]](https://github.com/Cadasta/ZipStream/issues/8)</sup>.

---

* [Use cases]
* [Supported backends]
  * [Database backend]
  * [Filestore backend]
* [Models]
  * [`FileRef`]
  * [`Bundle`]
* [Configuration]
* [API]
  * [One-off Zip Creation]
  * [Create Bundle]
  * [Download Bundle]
  * [Get Bundle Information]
  * [Update Bundle]
  * [Delete Bundle]
* [Getting Started]
* [Logging]
* [Code Coverage]
* [Docker]
* [FAQ]
* [Contributing]
* [Attributions]
* [History]
* [License]


---

## Use cases

### Assembling a bundle for later

Consider a situation where you store millions of files on Amazon S3 for a client. The client occasionally asks you to make a subset of their files available for download. ZipStream aims to simplify this by allowing you to submit references of the client's files to ZipStream and send your client a link the bundle's identifier. When your client visits the link, a zip file containing all of the S3 assets will be streamed to the user. Because of the fact that the service retrieves the remote data in small chunks, zips it, and streams it to users on the fly, the memory overhead is extremely low and zipfiles have no max filesize.

Naturally, if the zipped bundle is to be downloaded many times over, it's likely more efficient to actually generate the zipped bundle once, store it in a filestore, and send the client a URI to that zipped file. In this case, it would be better to use ZipStream's `/bundle` endpoint for a one-time generation of the to-be-shared zip file. This could be done efficiently via streaming from one service to the other.

<details>
 <summary>Example</summary>

```js
var AWS = require('aws-sdk');
var s3Stream = require('s3-upload-stream');
var request = require('request');

// Define s3-upload-stream with S3 credentials.
var awsConn = new AWS.S3({
  accessKeyId: '',
  secretAccessKey: ''
});

// Establish input stream
var inStream = request({
  url: 'https://myZipstreamServer.com/bundle',
  method: 'POST',
  json: {
    files: [
      {
        src: 'https://i.imgur.com/CMMdGGX.jpg',
        dst: 'rock-collection.jpg'
      }
    ]
  }
});

// Establish output stream
var outStream = s3Stream(awsConn)
  .upload({
    'Bucket': 'myBucket',
    'Key': 'my-key.zip'
  })
  .on('error', function (err) {
    console.error(err);
  });

// Send it
inStream.pipe(outStream)
```
</details>


### Bundling on the fly

Consider a situation where you would like to allow a customer to export a number of files of their choosing, perhaps in a shopping-cart fashion. Zipstream makes this simple by having a [one-off zip creation] endpoint that expects a `POST` request containing reference to externally hosted files and responds with a streamed zip of these files. Using this, a developer can set up a webform to enable users to select files of their liking and to initiate the download on submit.

## Supported backends

The system relies on two backend types: Database and Filestore.

### Database backend

The database backend is responsible for managing data that represent ZipStream [`Bundle`] instances.

To add support for a new database backend, a file should be placed in the `backends/db` directory and offer four functions: `create`, `read`, `update`, `delete`. Each function must take in a [`FileRef`] object and return a Promise object.

#### Currently supported database backends:

- [Amazon DynamoDB](https://aws.amazon.com/dynamodb/)
  - Requires `DB_INTERFACE` [config] variable to equal `'dynamodb'`.
  - Requires `TABLE_NAME` [config] variable to be set.
  - Requires `AWS_REGION` [config] variable to be set.
- [Postgresql](https://www.postgresql.org/)
  - Requires `DB_INTERFACE` [config] variable to equal `'postgres'`.
  - Requires `DB_CNXN` [config] variable to be set.
  - Requires `TABLE_NAME` [config] variable to be set.

To see what's on the database backends radar, see our [Issues](https://github.com/Cadasta/ZipStream/issues?q=is%3Aopen+label%3Aenhancement+label%database).

### Filestore backend

The filestore backend is responsible for returning files in a [Readable stream](https://nodejs.org/api/stream.html#stream_readable_streams) format. Multiple filestore backends may be enabled. When examining an submitted [`FileRef`], the system will look for a backend matching the protocol of the `src` attribute (e.g. a `src` of `s3://myBucket/my/key.jpg` would use the `s3` backend).

To add support for a new filestore backend, a file should be placed in the `backends/fs` directory and offer a single function: `getStream`. This function should take in a `src` argument and return a readable stream.

#### Currently supported filestore backends:

- [Amazon S3](https://aws.amazon.com/s3/)
  - Requires `FS_INTERFACES` [config] variable to contain `'s3'`.
  - Requires `AWS_REGION` [config] variable to be set.
  - Expects [`FileRef`] `src` attributes in the format of `s3://<bucketName>/<key>`.
- HTTP(S) - Any valid URL. _For security reasons, IP addresses are not supported_
  - Requires `FS_INTERFACES` [config] variable to contain `'https'`.
  - Expects [`FileRef`] `src` attributes in the format of `<http|https>://<path>`.

To see what's on the filestore backends radar, see our [Issues](https://github.com/Cadasta/ZipStream/issues?q=is%3Aopen+label%3Aenhancement+label%3Afilestore).


## Models

### `FileRef`

- `src`: `String`, protocol and location of file (depending on backend), such as S3 Bucket
- `dst`: `String`, _optional_, desired path of file when in bundle, defaults to path of `src` value

### `Bundle`

- `id`: `String`, UUIDv4
- `secret`: `String`, UUIDv4
- `files`: `Array`, [`FileRef`] objects
- `expirationDate`: `Number`, unix timestamp representing date at which record should be deleted.
- `filename`: `String`, desired filename of bundle (should end with .zip)


## Configuration

- `NODE_ENV`: `String`, environment mode of server. Defaults to `'development'`. ([read more](https://www.dynatrace.com/blog/the-drastic-effects-of-omitting-node_env-in-your-express-js-applications/))
- `PORT`: `Number`, port number for zipstream server. Defaults to `4040`.
- `DB_INTERFACE`: `String`, name of database backend to be used by zipstream instance. Defaults to `'postgres'`.
- `FS_INTERFACES`: `String`, comma-separated names of filestore backends to be supported by zipstream instance. Defaults to `'s3,https'`.
- `DB_CNXN`: `String`, [Connection string](https://github.com/vitaly-t/pg-promise/wiki/Connection-Syntax#connection-string) used to connect to Postgres database backend. Defaults to `'postgres://localhost:5432/zipstream'`.
- `AWS_REGION`: `String`, AWS Region to be used by S3 filestore backend and DynamoDB database backend. Defaults to `'us-west-2'`.
- `TABLE_NAME`: `String`, Name of table used by filestore backend. Defaults to `'bundles'`.
- `DATA_LIFETIME`: `Number`, Lifespan of `Bundle` record, in minutes.  Defaults to `10080` (1 week). Note that the codebase does not manage the removal of expired bundles. It is up to individual backends to set up expiration logic via TTL or database trigger settings


## API

### One-off Zip Creation

Returns a bundle of provided files.

**URL** : `/bundle`

**Method** : `POST`

<details>
  <summary>
    <b>Data constraints</b>
  </summary>

```json
{
    "filename": "[optional, filename of zip to be returned]",
    "files": [
        {
            "src": "[protocol and filestore-backend-related location of file]",
            "dst": "[optional, desired path of file when in bundle, defaults to path of `src` value]"
        }
    ]
}
```
</details>

<details>
 <summary>
  <b>Data example</b>
 </summary>

```json
{
    "files": [
        {
            "src": "s3://my-aws-bucket-1/path/to/foo.jpg",
            "dst": "foo.jpg"
        },
        {
            "src": "s3://some-other-bucket-2/bar.gif"
        }
    ]
}
```
</details>


#### Success Response


Streaming download of zipped bundle.

**Code** : `200 OK`

_Download of bundle containing `foo.jpg` and `bar.gif`._



### Create Bundle

Creates a bundle.

**URL** : `/`

**Method** : `POST`

<details>
  <summary>
    <b>Data constraints</b>
  </summary>

```json
{
    "filename": "[desired filename of bundle (should end with .zip)]"
}
```

Or, optionally:

```json
{
    "filename": "[desired filename of bundle]",
    "files": [
        {
            "src": "[protocol and filestore-backend-related location of file]",
            "dst": "[optional, desired path of file when in bundle, defaults to path of `src` value]"
        }
    ]
}
```
</details>

<details>
 <summary>
  <b>Data example</b>
 </summary>

```json
{
    "filename": "my-awesome-bundle.zip",
    "files": [
        {
            "src": "s3://my-aws-bucket-1/path/to/foo.jpg",
            "dst": "foo.jpg"
        },
        {
            "src": "s3://some-other-bucket-2/bar.gif"
        }
    ]
}
```
</details>


#### Success Response

JSON representation of created bundle.

**Code** : `201 CREATED`

<details>
  <summary>
    <b>Content example</b>
  </summary>

```json
{
   "expirationDate" : 1503029550,
   "secret" : "bd1be533-3c5c-4395-bda9-73dd288c5487",
   "filename" : "my-awesome-bundle.zip",
    "files": [
        {
            "src": "s3://my-aws-bucket-1/path/to/foo.jpg",
            "dst": "foo.jpg"
        },
        {
            "src": "s3://some-other-bucket-2/bar.gif"
        }
   ],
   "id" : "c4f6f218-afc4-4af1-ae1b-b22e9b058f26"
}
```
</details>

### Download Bundle

Download a bundle.

**URL** : `/:id/`

**Method** : `GET`

#### Success Response

Streaming download of zipped bundle.

**Code** : `200 OK`

_Download of `my-awesome-bundle.zip` containing `foo.jpg` and `bar.gif`._

### Get Bundle Information

Retrieves bundle information.

**URL** : `/:id/:secret/`

**Method** : `GET`

#### Success Response

JSON representation of retrieved bundle.

**Code** : `200 OK`

<details>
  <summary>
    <b>Content example</b>
  </summary>

```json
{
   "expirationDate" : 1503029550,
   "secret" : "bd1be533-3c5c-4395-bda9-73dd288c5487",
   "filename" : "my-awesome-bundle.zip",
    "files": [
        {
            "src": "s3://my-aws-bucket-1/path/to/foo.jpg",
            "dst": "foo.jpg"
        },
        {
            "src": "s3://some-other-bucket-2/bar.gif"
        }
   ],
   "id" : "c4f6f218-afc4-4af1-ae1b-b22e9b058f26"
}
```
</details>

### Update Bundle

Append files a bundle.

**URL** : `/:id/:secret/`

**Method** : `PUT`

<details>
  <summary>
    <b>Data constraints</b>
  </summary>

```json
{
    "files": [
        {
            "src": "[protocol and filestore-backend-related location of file]",
            "dst": "[optional, desired path of file when in bundle, defaults to path of `src` value]"
        }
    ]
}
```
</details>


<details>
 <summary>
  <b>Data example</b>
 </summary>

```json
{
    "filename": "my-awesome-bundle.zip",
    "files": [
        {
            "src": "s3://one-more-bucket-3/another/file.pdf",
            "dst": "another-one.pdf"
        }
    ]
}
```
</details>


#### Success Response

JSON representation of bundle with newly-appended data.

**Code** : `200 OK`

<details>
  <summary>
    <b>Content example</b>
  </summary>

```json
{
   "expirationDate" : 1503029550,
   "secret" : "bd1be533-3c5c-4395-bda9-73dd288c5487",
   "filename" : "my-awesome-bundle.zip",
    "files": [
        {
            "src": "s3://my-aws-bucket-1/path/to/foo.jpg",
            "dst": "foo.jpg"
        },
        {
            "src": "s3://some-other-bucket-2/bar.gif"
        },
        {
            "src": "s3://one-more-bucket-3/another/file.pdf",
            "dst": "another-one.pdf"
        }
   ],
   "id" : "c4f6f218-afc4-4af1-ae1b-b22e9b058f26"
}
```
</details>

### Delete Bundle

Used to create an empty bundle.

**URL** : `/:id/:secret/`

**Method** : `DELETE`

#### Success Response

JSON representation of deleted bundle.

**Code** : `200 OK`

<details>
  <summary>
    <b>Content example</b>
  </summary>

```json
{
   "expirationDate" : 1503029550,
   "secret" : "bd1be533-3c5c-4395-bda9-73dd288c5487",
   "filename" : "my-awesome-bundle.zip",
    "files": [
        {
            "src": "s3://my-aws-bucket-1/path/to/foo.jpg",
            "dst": "foo.jpg"
        },
        {
            "src": "s3://some-other-bucket-2/bar.gif",
        },
        {
            "src": "s3://one-more-bucket-3/another/file.pdf",
            "dst": "another-one.pdf"
        }
   ],
   "id" : "c4f6f218-afc4-4af1-ae1b-b22e9b058f26"
}
```
</details>


## Getting Started

Clone the repo:
```sh
git clone git@github.com:Cadasta/zipstream.git
cd zipstream
```

Install yarn:
```js
npm install -g yarn
```

Install dependencies:
```sh
yarn
```

Set environment (vars):
```sh
cp .env.example .env
```

Start server:
```sh
# Start server
yarn start

# Selectively set DEBUG env var to get logs
DEBUG=zipstream:* yarn start
```
Refer [debug](https://www.npmjs.com/package/debug) to know how to selectively turn on logs.


Tests:
```sh
# Run tests written in ES6
yarn test

# Run test along with enforced code coverage (configured via package.json)
yarn test:coverage

# Run tests on file change
yarn test:watch
```
_TODO: [Full test coverage](https://github.com/Cadasta/ZipStream/issues/9)_

Lint:
```sh
# Lint code with ESLint
yarn lint

# Run lint on any file change
yarn lint:watch
```

Other gulp tasks:
```sh
# Wipe out dist and coverage directory
gulp clean

# Default task: Wipes out dist and coverage directory. Compiles using babel.
gulp
```

##### Deployment

```sh
# compile to ES5
1. yarn build

# upload dist/ to your server
2. scp -rp dist/ user@dest:/path

# install production dependencies only
3. yarn --production

# Use any process manager to start your services
4. pm2 start dist/index.js
```

In production you need to make sure your server is always up so you should ideally use any of the process manager recommended [here](http://expressjs.com/en/advanced/pm.html).


## Logging

Universal logging library [winston](https://www.npmjs.com/package/winston) is used for logging. It has support for multiple transports.  A transport is essentially a storage device for your logs. Each instance of a winston logger can have multiple transports configured at different levels. For example, one may want error logs to be stored in a persistent remote location (like a database), but all logs output to the console or a local file. We just log to the console for simplicity, you can configure more transports as per your requirement.

#### API logging
Logs detailed info about each api request to console during development.
![Detailed API logging](https://cloud.githubusercontent.com/assets/4172932/12563354/f0a4b558-c3cf-11e5-9d8c-66f7ca323eac.JPG)

#### Error logging
Logs stacktrace of error to console along with other details. You should ideally store all error messages persistently.
![Error logging](https://cloud.githubusercontent.com/assets/4172932/12563361/fb9ef108-c3cf-11e5-9a58-3c5c4936ae3e.JPG)


## Code Coverage
Get code coverage summary on executing `yarn test`
![Code Coverage Text Summary](https://cloud.githubusercontent.com/assets/4172932/12827832/a0531e70-cba7-11e5-9b7c-9e7f833d8f9f.JPG)

`yarn test` also generates HTML code coverage report in `coverage/` directory. Open `lcov-report/index.html` to view it.
![Code coverage HTML report](https://cloud.githubusercontent.com/assets/4172932/12625331/571a48fe-c559-11e5-8aa0-f9aacfb8c1cb.jpg)


## Docker

[_TODO_](https://github.com/Cadasta/ZipStream/issues/4)


## FAQ

> Why isn't this written an AWS Lambda service?

This service would indeed be a good use case for AWS Lambda. In fact, we initially began building it out as a [Serverless](http://serverless.com) app. Ultimately, Lambda's 5 minute max-lifetime turned us off of the idea as we cater towards clients in remote, low-bandwidth regions where a 5+ minute download is likely. If you're interesting in running this codebase on AWS Lambda, [we'd love to hear how it goes!](https://github.com/Cadasta/ZipStream/issues/7)

## Contributing

Pull Requests are very welcome! If you would like to add a new feature, it is recommended that you create an Issue to first discuss the idea, however this is not mandatory.

## Attributions

Inspired by [Teamwork's s3zipper](https://github.com/Teamwork/s3zipper). Built from the [express-mongoose-es6-rest-api boilerplate](https://github.com/KunalKapadia/express-mongoose-es6-rest-api).

## History

For the list of all changes see the [CHANGELOG](CHANGELOG.md).

## License

[GNU Affero General Public License](LICENSE).

[`FileRef`]: #fileref
[`Bundle`]: #bundle

[config]: #configuration

[Use cases]: #use-cases
[Supported backends]: #supported-backends
[Database backend]: #database-backend
[Filestore backend]: #filestore-backend
[Models]: #models
[Configuration]: #configuration
[API]: #api
[One-off Zip Creation]: #one-off-zip-creation
[Create Bundle]: #create-bundle
[Download Bundle]: #download-bundle
[Get Bundle Information]: #get-bundle-information
[Update Bundle]: #update-bundle
[Delete Bundle]: #delete-bundle
[Getting Started]: #getting-started
[Logging]: #logging
[Code Coverage]: #code-coverage
[Docker]: #docker
[FAQ]: #faq
[Contributing]: #contributing
[Attributions]: #attributions
[History]: #history
[License]: #license
