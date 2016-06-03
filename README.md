# react-native-fetch-blob [![npm version](https://badge.fury.io/js/react-native-fetch-blob.svg)](https://badge.fury.io/js/react-native-fetch-blob) ![](https://img.shields.io/badge/PR-Welcome-brightgreen.svg)

## v0.5.0 Work In Progress README.md

Module for upload, download, and access files in JS context. Also provides file stream API for read/write large files.

If you're getting into trouble with image or file server that requires specific fields in the header, or you're having problem with `fetch` API when sending/receiving binary data, you might try this module as well.

See [[fetch] Does fetch with blob() marshal data across the bridge?](https://github.com/facebook/react-native/issues/854) for the reason why I made this module.

** Pre v0.5.0 Users**

This update is `backward-compatible` generally you don't have to change existing code unless you're going to use new APIs.

In latest version (v0.5.0), new APIs can either `upload` or `download` files simply using a file path. It's much more memory efficent in some use case. We've also introduced `fs` APIs for access files, and `file stream` API that helps you read/write files (especially for **large ones**), see [Examples](#user-content-usage) bellow.

This module implements native methods, supports both Android (uses awesome native library  [AsyncHttpClient](https://github.com/AsyncHttpClient/async-http-client])) and IOS.

## TOC

* [Installation](#user-content-installation)
* [Examples](#user-content-usage)
 * [Download file](#user-content-download-example--fetch-files-that-needs-authorization-token)
 * [Upload file](#user-content-upload-example--dropbox-files-upload-api)
 * [Multipart/form upload](#user-content-multipartform-data-example--post-form-data-with-file-and-data)
 * [Upload/Download progress](#user-content-uploaaddownload-progress)
 * [Show Downloaded File and Notification in Android Downloads App](#user-content-show-downloaded-file-in-android-downloads-app)
 * [File access](#user-content-file-access)
 * [File stream](#user-content-file-stream)
 * [Manage cached files](#user-content-manage-cached-files)
* [API](#user-content-api)
 * [config](#user-content-config)
 * [fetch](#user-content-fetchmethod-url-headers-bodypromisefetchblobresponse)
 * [session](#user-content-sessionnamestringrnfetchblobsession)
 * [base64](#user-content-base64)
 * [fs](#user-content-fs)
* [Development](#user-content-development)

## Installation

Install package from npm

```sh
npm install --save react-native-fetch-blob
```

Link package using [rnpm](https://github.com/rnpm/rnpm)

```sh
rnpm link
```

**Android Access Permission to External storage (Optional)**

If you're going to access external storage (say, SD card storage), you might have to add the following line to `AndroidManifetst.xml`.

```diff
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.rnfetchblobtest"
    android:versionCode="1"
    android:versionName="1.0">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
+   <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />                                               
+   <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />                                              

    ...

```

## Usage

```js
import RNFetchBlob from 'react-native-fetch-blob'
```
#### Download example : Fetch files that needs authorization token

```js

// send http request in a new thread (using native code)
RNFetchBlob.fetch('GET', 'http://www.example.com/images/img1.png', {
    Authorization : 'Bearer access-token...',
    // more headers  ..
  })
  // when response status code is 200
  .then((res) => {
    // the conversion is done in native code
    let base64Str = res.base64()
    // the following conversions are done in js, it's SYNC
    let text = res.text()
    let json = res.json()

  })
  // Status code is not 200
  .catch((errorMessage, statusCode) => {
    // error handling
  })
```

#### Download to storage directly

The simplest way is give a `fileCach` option to config, and set it to `true`. This will let the incoming response data stored in a temporary path **wihout** any file extension. 

**These files won't be removed automatically, please refer to [Cache File Management](#user-content-cache-file-management)**

```js
RNFetchBlob
  .config({
    // add this option that makes response data to be stored as a file,
    // this is much more performant.
    fileCache : true,
  })
  .fetch('GET', 'http://www.example.com/file/example.zip', {
    some headers ..
  })
  .then((res) => {
    // the temp file path
    console.log('The file saved to ', res.path())
  })
```

**Set Temp File Extension**

Sometimes you might need a file extension for some reason. For instance, when using file path as source of `Image` component, the path should end with something like .png or .jpg, you can do this by add `appendExt` option to `config`.

```js
RNFetchBlob
  .config({
    fileCache : true,
    // by adding this option, the temp files will have a file extension
    appendExt : 'png'
  })
  .fetch('GET', 'http://www.example.com/file/example.zip', {
    some headers ..
  })
  .then((res) => {
    // the temp file path with file extension `png`
    console.log('The file saved to ', res.path())
    // Beware that when using a file path as Image source on Android,
    // you must prepend "file://"" before the file path
    imageView = <Image source={{ uri : Platform.OS === 'android' ? 'file://' : '' + res.path() }}/>
  })
```

**Use Specific File Path**

If you prefer a specific path rather than random generated one, you can use `path` option. We've added a [getSystemDirs](#user-content-getsysdirs) API in v0.5.0 that lists several common used directories.

```js
RNFetchBlob.getSystemDirs().then((dirs) => {
  RNFetchBlob
    .config({
      // response data will be saved to this path if it has access right.
      path : dirs.DocumentDir + '/path-to-file.anything'
    })
    .fetch('GET', 'http://www.example.com/file/example.zip', {
      //some headers ..
    })
    .then((res) => {
      // the path should be dirs.DocumentDir + 'path-to-file.anything'
      console.log('The file saved to ', res.path())
    })
})
```

**These files won't be removed automatically, please refer to [Cache File Management](#user-content-cache-file-management)**

####  Upload example : Dropbox [files-upload](https://www.dropbox.com/developers/documentation/http/documentation#files-upload) API

`react-native-fetch-blob` will convert the base64 string in `body` to binary format using native API, this process will be  done in a new thread, so it's async.

```js

RNFetchBlob.fetch('POST', 'https://content.dropboxapi.com/2/files/upload', {
    Authorization : "Bearer access-token...",
    'Dropbox-API-Arg': JSON.stringify({
      path : '/img-from-react-native.png',
      mode : 'add',
      autorename : true,
      mute : false
    }),
    'Content-Type' : 'application/octet-stream',
    // here's the body you're going to send, should be a BASE64 encoded string 
    // (you can use "base64" APIs to make one). 
    // The data will be converted to "byte array"(say, blob) before request sent.  
  }, base64ImageString)
  .then((res) => {
    console.log(res.text())
  })
  .catch((err) => {
    // error handling ..
  })
```

#### Upload a file from storage

If you're going to use a `file` request body, just wrap the path with `wrap` API.

```js
RNFetchBlob.fetch('POST', 'https://content.dropboxapi.com/2/files/upload', {
    // dropbox upload headers
    Authorization : "Bearer access-token...",
    'Dropbox-API-Arg': JSON.stringify({
      path : '/img-from-react-native.png',
      mode : 'add',
      autorename : true,
      mute : false
    }),
    'Content-Type' : 'application/octet-stream',
    // Change BASE64 encoded data to a file path with prefix `RNFetchBlob-file://` when the data comes from a file.
  }, RNFetchBlob.wrap(PATH_TO_THE_FILE))
  .then((res) => {
    console.log(res.text())
  })
  .catch((err) => {
    // error handling ..
  })
```

#### Multipart/form-data example : Post form data with file and data

In `version >= 0.3.0` you can also post files with form data, just put an array in `body`, with elements have property `name`, `data`, and `filename`(optional).

Elements have property `filename` will be transformed into binary format, otherwise it turns into utf8 string.

```js

  RNFetchBlob.fetch('POST', 'http://www.example.com/upload-form', {
    Authorization : "Bearer access-token",
    otherHeader : "foo",
    'Content-Type' : 'multipart/form-data',
  }, [
    // element with property `filename` will be transformed into `file` in form data
    { name : 'avatar', filename : 'avatar.png', data: binaryDataInBase64},
    // elements without property `filename` will be sent as plain text
    { name : 'name', data : 'user'},
    { name : 'info', data : JSON.stringify({
      mail : 'example@example.com',
      tel : '12345678'
    })},
  ]).then((resp) => {
    // ...
  }).catch((err) => {
    // ...
  })
```

What if you want to upload a file in some field ? Just like [upload a file from storage](#user-content-upload-a-file-from-storage) example, wrap `data` by `wrap` API (this feature is only available for `version >= v0.5.0`)

```js

  RNFetchBlob.fetch('POST', 'http://www.example.com/upload-form', {
    Authorization : "Bearer access-token",
    otherHeader : "foo",
    // this is required, otherwise it won't be process as a multipart/form-data request
    'Content-Type' : 'multipart/form-data',
  }, [
    // append field data from file path
    { 
      name : 'avatar',
      filename : 'avatar.png',
      // Change BASE64 encoded data to a file path with prefix `RNFetchBlob-file://` when the data comes from a file path
      data: RNFetchBlob.wrap(PATH_TO_THE_FILE)
    },
    // elements without property `filename` will be sent as plain text
    { name : 'name', data : 'user'},
    { name : 'info', data : JSON.stringify({
      mail : 'example@example.com',
      tel : '12345678'
    })},
  ]).then((resp) => {
    // ...
  }).catch((err) => {
    // ...
  })
```

#### Upload/Download progress

In `version >= 0.4.2` it is possible to know the upload/download progress.

```js
  RNFetchBlob.fetch('POST', 'http://www.example.com/upload', {
      ... some headers,
      'Content-Type' : 'octet-stream'
    }, base64DataString)
    .progress((received, total) => {
        console.log('progress', received / total)
    })
    .then((resp) => {
      // ...
    })
    .catch((err) => {
      // ...
    })
```

#### Show Downloaded File and Notifiction in Android Downloads App

When you use `config` API to store response data to file, the file won't be visible in Andoird's "Download" app, if you want to do this, some extra options in `config` is required.

```js
RNFetchBlob.config({
  fileCache : true,
  // android only options
  addAndroidDownloads : {
    // Show notification when response data transmitted
    notification : true,
    // Title of download notification
    title : 'Great ! Download Success ! :O ',
    // File description (not notification description)
    description : 'An image file.',
    mime : 'image/png',
    // Make the file scannable  by media scanner
    meidaScannable : true,
  }
})
.fetch('GET', 'http://example.com/image1.png')
.then(...)
```

#### File Access

File access APIs were made when developing `v0.5.0`, which helping us write tests, and was not planned to be a part of this module. However I realized that, it's hard to find a great solution to manage cached files, every one who use this moudle may need those APIs for there cases. 

Here's the list of `fs` APIs

- getSystemDirs
- createFile
- readStream
- writeStream
- unlink
- mkdir
- ls
- mv
- cp
- exists
- isDir

See [fs](#user-content-fs) chapter for more information

#### File Stream

In `v0.5.0` we've added  `writeStream` and `readStream`, which allows your app read/write data from file path. This API creates a file stream, rather than convert whole data into BASE64 encoded string, it's handy when processing **large files**.

But there're some differences between `readStream` and `writeStream` API. When calling `readStream` method, the file stream is opened immediately, and start to read data. 

```js
let data = ''
let ifstream = RNFetchBlob.readStream(
    // encoding, should be one of `base64`, `utf8`
    'base64',
    // file path
    PATH_TO_THE_FILE,
    // (optional) buffer size, default to 4096 (4095 for BASE64 encoded data)
    // when reading file in BASE64 encoding, buffer size must be multiples of 3.
    4095)
ifstream.onData((chunk) => {
  data += chunk
})
ifstream.onError((err) => {
  console.log('oops', err)
})
ifstream.onEnd(() => {  
  <Image source={{ uri : 'data:image/png,base64' + data }}
})
```

When use `writeStream`, the stream is also opened immediately, but you have to `write`, and `close` by yourself.

```js
let ofstream = RNFetchBlob.writeStream(
    PATH_TO_FILE, 
    // encoding, should be one of `base64`, `utf8`, `ascii`
    'utf8',
    // should data append to existing content ?
    true)
ofstream.write('foo')
ofstream.write('bar')
ofstream.close()

```

#### Cache File Management

When using `fileCache` or `path` options along with `fetch` API, response data will automatically stored into file system. The files will **NOT** removed unless you `unlink` it. There're several ways to remove the files

```js

  // remove file using RNFetchblobResponse.flush() object method
  RNFetchblob.config({
      fileCache : true
    })
    .fetch('GET', 'http://example.com/download/file')
    .then((res) => {
      // remove cached file from storage
      res.flush()
    })

  // remove file by specifying a path
  RNFetchBlob.unlink('some-file-path').then(() => {
    // ...
  })

```

You can also group the requests by using `session` API, and use `dispose` to remove them all when needed.

```js

  RNFetchblob.config({
    fileCache : true
  })
  .fetch('GET', 'http://example.com/download/file')
  .then((res) => {
    // set session of a response
    res.session('foo')
  })  

  RNFetchblob.config({
    // you can also set session beforehand
    session : 'foo'
    fileCache : true
  })
  .fetch('GET', 'http://example.com/download/file')
  .then((res) => {
    // ...
  })  

  // or put an existing file path to the session
  RNFetchBlob.session('foo').add('some-file-path')
  // remove a file path from the session
  RNFetchBlob.session('foo').remove('some-file-path')
  // list paths of a session
  RNFetchBlob.session('foo').list()
  // remove all files in a session
  RNFetchBlob.session('foo').dispose().then(() => { ... })

```

## API

#### `config(options:RNFetchBlobConfig):fetch`

`0.5.0`

Config API was introduced in `v0.5.0` which provides some options for the `fetch` task. 

#### `fetch(method, url, headers, body):Promise<FetchBlobResponse>`

`legacy`

Send a HTTP request uses given headers and body, and return a Promise.

#### method:`string` Required
HTTP request method, can be one of `get`, `post`, `delete`, and `put`, case-insensitive.
#### url:`string` Required
HTTP request destination url.
#### headers:`object` (Optional)
Headers of HTTP request, value of headers should be `stringified`, if you're uploading binary files, content-type should be `application/octet-stream` or `multipart/form-data`(see examples above).
#### body:`string | Array<Object>` (Optional)
Body of the HTTP request, body can either be a BASE64 string, or an array contains object elements, each element have 2  required property `name`, and `data`, and 1 optional property `filename`, once `filename` is set, content in `data` property will be consider as BASE64 string that will be converted into byte array later.
When body is a base64 string , this string will be converted into byte array in native code, and the request body will be sent as `application/octet-stream`.

#### `fetch(...).progress(eventListener):Promise<FetchBlobResponse>`

`0.4.2`

Register on progress event handler for a fetch request.

#### eventListener:`(sendOrReceivedBytes:number, totalBytes:number)`

A function that triggers when there's data received/sent, first argument is the number of sent/received bytes, and second argument is expected total bytes number.

#### `session(name:string):RNFetchBlobSession`

TODO

#### `base64`

`0.4.2`

A helper object simply uses [base-64](https://github.com/mathiasbynens/base64) for decode and encode BASE64 data.

```js
RNFetchBlob.base64.encode(data)
RNFetchBlob.base64.decode(data)
```

#### `fs`

`0.5.0`

#### `getSystemDirs():Promise<object>`

This method returns common used folders:

- DocumentDir 
- CacheDir
- DCIMDir (Android Only)
- DownloadDir (Android Only)

example 

```js
RNFetchBlob.getSystemDirs().then((dirs) => {
    console.log(dirs.DocumentDir)
    console.log(dirs.CacheDir)
    console.log(dirs.DCIMDir)
    console.log(dirs.DownloadDir)
})
```

If you're going to make downloaded file visible in Android `Downloads` app, please see [Show Downloaded File and Notification in Android Downloads App](#user-content-show-downloaded-file-in-android-downloads-app).

#### createFile(path:string, data:string, encoding:string ):Promise

TODO

#### writeStream(path:string, encoding:string, append:boolean):Promise<WriteStream>

TODO

#### readStream(path:string, encoding:string, append:boolean):Promise<ReadStream>

TODO

#### mkdir(path:string):Promise

TODO

#### ls(path:string):Promise

TODO

#### mv(from:string, to:string):Promise

TODO

#### cp(path:string):Promise

TODO

#### exists(path:string):Promise<boolean>

TODO

#### isDir(path:string):Promise<boolean>

TODO

#### unlink(path:string):Promise<boolean>

TODO

### Types

#### RNFetchBlobConfig

A set of configurations that will be injected into a `fetch` method, with the following properties.

#### fileCache:boolean
  Set this property to `true` will makes response data of the `fetch` stored in a temp file, by default the temp file will stored in App's own root folder with file name template `RNFetchBlob_tmp${timestamp}`.
#### appendExt:string
  Set this propery to change temp file extension that created by `fetch` response data.
#### path:string
  When this property has value, `fetch` API will try to store response data in the path ignoring `fileCache` and `appendExt` property.
#### addAndroidDownloads:object (Android only)
  This is an Android only property, it should be an object with the following properties :
  - title : title of the file download success notification
  - description : File description of the file.
  - mime : MIME type of the file. By default is `text/plain`
  - mediaScannable : A `boolean` value, see [Officail Document](https://developer.android.com/reference/android/app/DownloadManager.html#addCompletedDownload(java.lang.String, java.lang.String, boolean, java.lang.String, java.lang.String, long, boolean))
  - notification : A `boolean` value decide whether show a notification when download complete.
  

#### RNFetchBlobResponse

When `fetch` success, it resolve a `FetchBlobResponse` object as first argument. `FetchBlobResponse` object has the following methods (these method are synchronous, so you might take quite a performance impact if the file is big)

#### base64():string
  returns base64 string of response data (done in native context)
#### json():object
  returns json parsed object (done in js context)
#### text():string
  returns decoded base64 string (done in js context)
#### path():string
  returns file path if the response data is cached in file
#### session(name:string):RNFetchBlobSession
  when the response data is cached in a file, this method adds the file into the session. The following usages are equivalent.
```js
RNFetchBlob.session('session-name').add(resp.path())
// or
resp.session('session-name')
```

#### RNFetchBlobSession

A `session` is an object that helps you manage files. It simply main a list of file path and let you use `dispose()`to delete files in this session once and for all.

#### add(path:string):RNFetchBlobSession
  Add a file path to this session.
#### remove(path:string):RNFetchBlobSession
  Remove a file path from this session (not delete the file).
#### list():Array<String>
  Returns an array contains file paths in this session.
#### dispose():Promise
  Delete all files according to paths in the session.

## Major Changes

| Version | |
|---|---|
| ~0.3.0 | Upload/Download octet-stream and form-data |
| 0.4.0 | Add base-64 encode/decode library and API |
| 0.4.1 | Fixe upload form-data missing file extension problem on Android |
| 0.4.2 | Supports upload/download progress |
| 0.5.0 | Upload/download with direct access to file storage, and also added file access APIs |

### TODOs

* Customizable Multipart MIME type
* Improvement of file cache management API

### Development

If you're interested in hacking this module, check our [development guide](https://github.com/wkh237/react-native-fetch-blob/wiki/Development-Guide), there might be some helpful information.
Please feel free to make a PR or file an issue.
