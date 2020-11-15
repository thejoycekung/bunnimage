## Step 2: Convert The Image 🔄

### Create another Azure Function
Yep... We need yet *another* Azure Function. (What can I say? They're pretty helpful.) This one will trigger when **the image blob is stored**, convert it into a PDF, and store it in to the "pdfs" container.

⚠😵**WARNING**😵⚠ Lots of code ahead, but it's all good! I split it into sections.

```js
var multipart = require("parse-multipart");
var fetch = require("node-fetch");
const { BlobServiceClient } = require("@azure/storage-blob");
const connectionstring = process.env["AZURE_STORAGE_CONNECTION_STRING"];
const account = "bunnimagestorage";

module.exports = async function (context, eventGridEvent, inputBlob) {
    context.log("<----------------START----------------------->")
    
    const blobUrl = context.bindingData.data.url;
    const blobName = blobUrl.slice(blobUrl.lastIndexOf("/")+1);
    // get blob url and name
    context.log(`Blob Url: ${blobUrl}`);
    //context.log(eventGridEvent);
    context.log(`Image that was uploaded: ${blobName}`);
    
    var result = await convertImage(blobName);
    jobId = result.id

    status = false;
    try {
        while (!status) {
            update = await checkStatus(jobId);
            context.log("<------Trying-------->")
            context.log(update);
            if (update.status.code == "completed") {
                context.log("[!] Image done converting")
                status = true
                uri = update.output[0].uri;
                filename = update.output[0].filename;
                context.log("Download URL: " + uri);
                context.log("Donwnload file: " + filename);
            } else if (update.status.info == "The file has not been converted due to errors."){
                context.log("[!!!] AccountKey is incorrect")
                context.done();
            }
        }
    }
    catch (e) {
        context.log("[!!!] Daily Conversions Reached");
        context.done();
    }

    //url = "https://www22.online-convert.com/dl/web7/download-file/ba0bc036-73a6-4bce-9230-1d0394224da1/thumbn.pdf"

    let resp = await fetch(uri, {
        method: 'GET',
    })

    let data = await resp.buffer();
    context.log(data);
    // let newfile = await uploadBlob(data, filename);
    const uploadStatus = await uploadBlob(data, filename);
    context.log("[!] Uploading to pdfs container...")
    context.log(uploadStatus);
    context.log("<----------------END---------------------->")
    context.done();
};
```
```js
async function convertImage(blobName){
    const api_key = process.env['convertAPI_KEY'];
    const accountKey = process.env['accountKey'];
    const uriBase = "https://api2.online-convert.com/jobs";
	// env variables (similar to .gitignore/.env file) to not expose personal info

    img = {
    "conversion": [{
        "target": "pdf"
    }],
    "input": [{
        "type": "cloud",
        "source": "azure",
        "parameters": {
            "container": "images",
            "file": blobName
        },
        "credentials": {
            "accountname": "bunnimagestorage",
            "accountkey": accountKey
        }
    }]
    }

    payload = JSON.stringify(img);

    // making the post request
    let resp = await fetch(uriBase, {
        method: 'POST',
        body: payload,
        // we want to send the image
        headers: {
            'x-oc-api-key' : api_key,
            'Content-type' : 'application/json',
            'Cache-Control' : 'no-cache'
        }
    })

    // receive the response
    let data = await resp.json();

    return data;
}
```
```js
async function checkStatus(jobId){
    const api_key = process.env['convertAPI_KEY'];
    const uriBase = "https://api2.online-convert.com/jobs";
	// env variables (similar to .gitignore/.env file) to not expose personal info

    // making the post request
    let resp = await fetch(uriBase + "/" + jobId, {
        /*The await expression causes async function execution to pause until a Promise is settled 
        (that is, fulfilled or rejected), and to resume execution of the async function after fulfillment. 
        When resumed, the value of the await expression is that of the fulfilled Promise*/
        method: 'GET',
        headers: {
            'x-oc-api-key' : api_key,
        }
    })

    // receive the response
    let data = await resp.json();

    return data;
}
```
```js
async function uploadBlob(pdf, filename){
    // create blobserviceclient object that is used to create container client
    const blobServiceClient = await BlobServiceClient.fromConnectionString(connectionstring);
    // get reference to a container
    const container = "pdfs";
    const containerClient = await blobServiceClient.getContainerClient(container);
    // create blob name
    const blobName = filename;
    // get block blob client
    const blockBlobClient = containerClient.getBlockBlobClient(blobName);
    const uploadBlobResponse = await blockBlobClient.upload(pdf, pdf.length);
    console.log(`Upload block blob ${blobName} successfully`, uploadBlobResponse.requestId);
    result = {
        body : {
            name : blobName, 
            type: pdf.type,
            data: pdf.length,
            success: true
        }
    };
    return result;
}
```

### Creating an Event Subscription
When the image blob is stored in the "images" container, we want the conversion from jpg/jpeg/png to pdf to begin *immediately*! 
> Just like when you "subscribe" to a YouTube channel it gives you notifications, we're going to subscribe to our own Blob Storage and trigger the Azure Function.

**1. Search "Event Grid Subscriptions" in the search bar**

**2. Click "+ Event Subscription" in the top left**

**3. Actually creating it:**

![image](https://user-images.githubusercontent.com/69332964/99189683-5c934180-2730-11eb-8451-17762bf46866.png)
* If it asks you for a name, feel free to put anything you want (make it make sense though...)
* Topic Types = Storage Accounts
* Resource Group = [Insert your Resource Group that holds your storage account]
* Resource = [Insert your storage account name]

*Note: If your storage account doesn't appear, you forgot to follow the "upgrade to v2 storage" step*
* Event Types: **Only select Blob Created**

![image](https://user-images.githubusercontent.com/69332964/99189740-aed46280-2730-11eb-8ff0-c8a7ba19aadc.png)
* Endpoint Type: Azure Function

![image](https://user-images.githubusercontent.com/69332964/99189763-d0354e80-2730-11eb-91e4-5b17fc5e63bd.png)
* Function = [The one we just created to convert the image.. NOT the one that uploads the image blob]

**4. Tweaking some settings...**

* Navigate to the "Filters" tab and "Enable Subject Filtering"
![image](https://user-images.githubusercontent.com/69332964/99189929-bd6f4980-2731-11eb-9b01-b0cef972b96a.png)
* Change the "Subject Begins With" to `/blobServices/default/containers/images/blobs/`
  * This way, the subscription will **not** trigger when a PDF is stored in the "pdfs" container. It will **only** trigger when something is stored in "images."

> Congratulations! You have now subscribed to the "blob created" event in your "images" container that triggers the convert image function!