# Lab: Automate resizing uploaded images using Event Grid

## Overwiev

In this scenario we will add serverless automatic thumbnail generation using Azure Event Grid and Azure Functions. Event Grid enables Azure Functions to respond to Azure Blob storage events and generate thumbnails of uploaded images. An event subscription is created against the Blob storage create event. When a blob is added to a specific Blob storage container, a function endpoint is called. Data passed to the function binding from Event Grid is used to access the blob and generate the thumbnail image.

## Task 1: Run Cloud Shell and register Event Grid resource provider


1. Start Cloud Shell.
2. Run command: ```az provider register --namespace Microsoft.EventGrid``` to register Event Grid resource provider.

## Task 2: Create Azure Blob Storage and Azure Function
1. Create an Azure Blob Storage from Azure Portal in West Europe region.
2. Create Azure Function.
* Set runtime to NodeJS 12LTS
3. Create new Function called Thumbnail, triggered by EventGrid.


## Task 3: Insert application code:

1. Open function code.
2. Enter code to index.js file:

```
const stream = require('stream');
const Jimp = require('jimp');

const {
  Aborter,
  BlobURL,
  BlockBlobURL,
  ContainerURL,
  ServiceURL,
  SharedKeyCredential,
  StorageURL,
  uploadStreamToBlockBlob
} = require("@azure/storage-blob");

const ONE_MEGABYTE = 1024 * 1024;
const ONE_MINUTE = 60 * 1000;
const uploadOptions = { bufferSize: 4 * ONE_MEGABYTE, maxBuffers: 20 };

const containerName = process.env.BLOB_CONTAINER_NAME;
const accountName = process.env.AZURE_STORAGE_ACCOUNT_NAME;
const accessKey = process.env.AZURE_STORAGE_ACCOUNT_ACCESS_KEY;

const sharedKeyCredential = new SharedKeyCredential(
  accountName,
  accessKey);
const pipeline = StorageURL.newPipeline(sharedKeyCredential);
const serviceURL = new ServiceURL(
  `https://${accountName}.blob.core.windows.net`,
  pipeline
);

module.exports = (context, eventGridEvent, inputBlob) => {  

  const aborter = Aborter.timeout(30 * ONE_MINUTE);
  const widthInPixels = 100;
  const contentType = context.bindingData.data.contentType;
  const blobUrl = context.bindingData.data.url;
  const blobName = blobUrl.slice(blobUrl.lastIndexOf("/")+1);

  const image = await Jimp.read(inputBlob);
  const thumbnail = image.resize(widthInPixels, Jimp.AUTO);
  const thumbnailBuffer = await thumbnail.getBufferAsync(Jimp.AUTO);
  const readStream = stream.PassThrough();
  readStream.end(thumbnailBuffer);

  const containerURL = ContainerURL.fromServiceURL(serviceURL, containerName);
  const blockBlobURL = BlockBlobURL.fromContainerURL(containerURL, blobName);
  try {

    await uploadStreamToBlockBlob(aborter, readStream,
      blockBlobURL, uploadOptions.bufferSize, uploadOptions.maxBuffers,
      { blobHTTPHeaders: { blobContentType: "image/*" } });

  } catch (err) {

    context.log(err.message);

  } finally {

    context.done();

  }
};
```
3. In portal click Functions->Thumbnail->Integration.
4. Create new EventGrid Subsription with topic name: imagestoragesystopic
5. Switch to the Filters tab, and do the following actions:
* Select Enable subject filtering option.
* For Subject begins with, enter the following value : /blobServices/default/containers/images/blobs/.
6. Select Create to add the event subscription. This creates an event subscription that triggers the Thumbnail function when a blob is added to the images container. The function resizes the images and adds them to the thumbnails container.
