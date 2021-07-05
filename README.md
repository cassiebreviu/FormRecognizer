# Extract Data from PDFs using Form Recognizer with Code or Without!

Form recognizer is a powerful tool to help build a variety of document machine learning solutions.  It is one service however its made up of many prebuilt models that can perform a variety of essential document functions. You can even custom train a model using supervised or unsupervised learning for tasks outside of the scope of the prebuilt models! Read more about all the features of Form Recognizer [here](https://docs.microsoft.com/azure/cognitive-services/form-recognizer/overview?WT.mc_id=aiml-14201-cassieb). In this example we will be looking at how to use one of the prebuilt models in the Form Recognizer service that can extract the data from a PDF document dataset. Our documents are invoices with common data fields so we are able to use the prebuilt model without having to build a customized model.

Sample Invoice:
![img](/imgs/invoice.png)

After we take a look at how to do this with Python and Azure Form Recognizer, we will take a look at how to do the same process with no code using the Power Platform services: Power Automate and Form Recognizer built into AI Builder. In the Power Automate flow we are scheduling a process to happen every day. What the process does is look in the `raw` blob container to see if there is new files to be processed. If there is new files to be processed it gets all blobs from the container and loops through each blob to extract the PDF data using a prebuilt AI builder step. Then it deletes the processed document from the `raw` container. See what it looks like below:

Power Automate Flow:

![img](/imgs/flowaibuild.png)

### Prerequisites for Python
-	Azure Account [Sign up here!](https://azure.microsoft.com/free/?WT.mc_id=aiml-14201-cassieb)
-  [Anaconda](https://www.anaconda.com/products/individual) and/or [VS Code](https://code.visualstudio.com/Download?WT.mc_id=aiml-14201-cassieb)
-  Basic programming knowledge

### Prerequisites for Power Automate
- Power Automate Account [Sign up here!](https://docs.microsoft.com/power-automate/sign-up-sign-in/?WT.mc_id=aiml-14201-cassieb)
- No programming knowledge

## Process PDFs with Python and Azure Form Recognizer Service

### Create Services
First lets create the Form Recognizer Cognitive Service.
-	Go to portal.azure.com to create the resource or click this [link](https://ms.portal.azure.com/?WT.mc_id=aiml-0000-cassieb#create/Microsoft.CognitiveServicesFormRecognizer).

Now lets create a storage account to store the PDF dataset we will be using in containers. We want two containers, one for the `processed` PDFs and one for the `raw` unprocessed PDF.
-	Create an [Azure Storage Account](https://docs.microsoft.com/azure/storage/common/storage-account-create?WT.mc_id=aiml-14201-cassieb)
- Create two containers: `processed`, `raw`

### Upload data
Upload your dataset to the Azure Storage `raw` folder since they need to be processed. Once processed then they would get moved to the `processed` container.

The result should look something like this:

![img](/imgs/storageaccounts.png)


### Create Notebook and Install Packages
Now that we have our data stored in Azure Blob Storage we can connect and process the PDF forms to extract the data using the Form Recognizer Python SDK. You can also use the Python SDK with local data if you are not using Azure Storage. This example will assume you are using Azure Storage.

- Create a new [Jupyter notebook in VS Code](https://code.visualstudio.com/docs/python/jupyter-support?WT.mc_id=aiml-0000-cassieb#_create-or-open-a-jupyter-notebook?WT.mc_id=aiml-14201-cassieb).

- Install the Python SDK
```python
!pip install azure-ai-formrecognizer --pre
```

- Then we need to import the packages.

```python
import os
from azure.core.exceptions import ResourceNotFoundError
from azure.ai.formrecognizer import FormRecognizerClient
from azure.core.credentials import AzureKeyCredential
import os, uuid
from azure.storage.blob import BlobServiceClient, BlobClient, ContainerClient, __version__
```
### Create FormRecognizerClient
- Update the `endpoint` and `key` with the values from the service you created. These values can be found in the Azure Portal under the Form Recongizer service you created under the `Keys and Endpoint` on the navigation menu.
```python 
endpoint = "<your endpoint>"
key = "<your key>"
```

- We then use the `endpoint` and `key` to connect to the service and create the [FormRecongizerClient](https://docs.microsoft.com/python/api/azure-ai-formrecognizer/azure.ai.formrecognizer.aio.formrecognizerclient?WT.mc_id=aiml-14201-cassieb)
```python
form_recognizer_client = FormRecognizerClient(endpoint, AzureKeyCredential(key))
```
- Create the `print_results` helper function for use later to print out the results of each invoice.
```python
def print_result(invoices, blob_name):
    for idx, invoice in enumerate(invoices):
        print("--------Recognizing invoice {}--------".format(blob_name))
        vendor_name = invoice.fields.get("VendorName")
        if vendor_name:
            print("Vendor Name: {} has confidence: {}".format(vendor_name.value, vendor_name.confidence))
        vendor_address = invoice.fields.get("VendorAddress")
        if vendor_address:
            print("Vendor Address: {} has confidence: {}".format(vendor_address.value, vendor_address.confidence))
        customer_name = invoice.fields.get("CustomerName")
        if customer_name:
            print("Customer Name: {} has confidence: {}".format(customer_name.value, customer_name.confidence))
        customer_address = invoice.fields.get("CustomerAddress")
        if customer_address:
            print("Customer Address: {} has confidence: {}".format(customer_address.value, customer_address.confidence))
        customer_address_recipient = invoice.fields.get("CustomerAddressRecipient")
        if customer_address_recipient:
            print("Customer Address Recipient: {} has confidence: {}".format(customer_address_recipient.value, customer_address_recipient.confidence))
        invoice_id = invoice.fields.get("InvoiceId")
        if invoice_id:
            print("Invoice Id: {} has confidence: {}".format(invoice_id.value, invoice_id.confidence))
        invoice_date = invoice.fields.get("InvoiceDate")
        if invoice_date:
            print("Invoice Date: {} has confidence: {}".format(invoice_date.value, invoice_date.confidence))
        invoice_total = invoice.fields.get("InvoiceTotal")
        if invoice_total:
            print("Invoice Total: {} has confidence: {}".format(invoice_total.value, invoice_total.confidence))
        due_date = invoice.fields.get("DueDate")
        if due_date:
            print("Due Date: {} has confidence: {}".format(due_date.value, due_date.confidence))

```
### Connect to Blob Storage

- Now lets [connect to our blob storage containers](https://docs.microsoft.com/azure/storage/blobs/storage-quickstart-blobs-python?WT.mc_id=aiml-14201-cassieb) and create the [BlobServiceClient](https://docs.microsoft.com/python/api/azure-storage-blob/azure.storage.blob.blobserviceclient?WT.mc_id=aiml-14201-cassieb). We will use the client to connect to the `raw` and `processed` containers that we created earlier.

```python
# Create the BlobServiceClient object which will be used to get the container_client
connect_str = "<Get connection string from the Azure Portal>"
blob_service_client = BlobServiceClient.from_connection_string(connect_str)

# Container client for raw container.
raw_container_client = blob_service_client.get_container_client("raw")

# Container client for processed container
processed_container_client = blob_service_client.get_container_client("processed")

# Get base url for container.
invoiceUrlBase = raw_container_client.primary_endpoint
print(invoiceUrlBase)
```

*HINT: If you get a "HttpResponseError: (InvalidImageURL) Image URL is badly formatted." error make sure the proper permissions to access the container are set. Learn more about Azure Storage Permissions [here](https://docs.microsoft.com/azure/storage/common/storage-auth?WT.mc_id=aiml-14201-cassieb)*


### Extract Data from PDFs

We are ready to process the blobs now! Here we will call `list_blobs` to get a list of blobs in the `raw` container. Then we will loop through each blob, call the `begin_recognize_invoices_from_url` to extract the data from the PDF. Then we have our helper method to print the results. Once we have extracted the data from the PDF we will `upload_blob` to the `processed` folder and `delete_blob` from the raw folder.

```python
print("\nProcessing blobs...")

blob_list = raw_container_client.list_blobs()
for blob in blob_list:
    invoiceUrl = f'{invoiceUrlBase}/{blob.name}'
    print(invoiceUrl)
    poller = form_recognizer_client.begin_recognize_invoices_from_url(invoiceUrl)

    # Get results
    invoices = poller.result()

    # Print results
    print_result(invoices, blob.name)

    # Copy blob to processed
    processed_container_client.upload_blob(blob, blob.blob_type, overwrite=True)

    # Delete blob from raw now that its processed
    raw_container_client.delete_blob(blob)
```

Each result should look similar to this for the above invoice example:

![img](/imgs/pythonresult.png)

The prebuilt invoices model worked great for our invoices so we don't need to train a customized Form Recognizer model to improve our results.  But what if we did and what if we didn't know how to code?! You can still leverage all this awesomeness in AI Builder with Power Automate without writing any code. We will take a look at this same example in Power Automate next.

## Use Form Recognizer with AI Builder in Power Automate

You can achieve these same results using no code with Form Recognizer in AI Builder with Power Automate. Lets take a look at how we can do that.

### Create a New Flow
- Log in to [Power Automate](https://flow.microsoft.com/?WT.mc_id=aiml-0000-cassieb)
- Click `Create` then click `Instant Cloud Flow`. You can trigger Power Automate flows in a variety of ways so keep in mind that you may want to select a different trigger for you project.
- Give the Flow a name and select the `Manually trigger a flow`.

### Connect to Blob Storage

- Click `New Step`
- `List blobs` Step
    - Search for `Azure Blob Storage` and select `List blobs`
    - Select the ellipsis click `Create new connection` if your storage account isn't already connected
        - Fill in the `Connection Name`, `Azure Storage Account name` (the account you created), and the `Azure Storage Account Access Key` (which you can find in the resource keys in the Azure Portal)
        - Then select `Create`
    - Once the storage account is selected click the folder icon on the right of the list blobs options. You should see all the containers in the storage account, select `raw`.

Your flow should look something like this:
![img](/imgs/connecttoblob.png)

### Loop Through Blobs to Extract the Data
- Click the plus sign to create a new step
- Click `Control` then `Apply to each`
- Select the textbox and a list of blob properties will appear. Select the `value` property
- Next select `add action` from within the `Apply to each` Flow step.
- Add the `Get blob content` step:
    - Search for `Azure Blob Storage` and select `Get blob content`
    - Click the textbox and select the `Path` property. This will get the `File content` that we will pass into the Form Recognizer.
- Add the `Process and save information from invoices` step:
    - Click the plus sign and then `add new action`
    - Search for `Process and save information from invoices`
    - Select the textbox and then the property `File Content` from the `Get blob content` section
- Add the `Copy Blob` step:
    - Repeat the add action steps
    - Search for `Azure Blob Storage` and select `Copy Blob`
    - Select the `Source url` text box and select the `Path` property
    - Select the `Destination blob path` and put `/processed` for the processed container
    - Select `Overwrite?` dropdown and select `Yes` if you want the copied blob to overwrite blobs with the existing name.
- Add the `Delete Blob` step:
    - Repeat the add action steps
    - Search for `Azure Blob Storage` and select `Delete Blob`
    - Select the `Blob` text box and select the `Path` property

The `Apply to each` block should look something like this:
![img](/imgs/applytoeachblock.png)

- Save and Test the Flow
    - Once you have completed creating the flow save and test it out using the built in test features that are part of power automate.

This prebuilt model again worked great on our invoice data. However if you have a more complex dataset, use the AI Builder to label and create a customized machine learning model for your specific dataset. Read more about how to do that [here](https://docs.microsoft.com/azure/cognitive-services/form-recognizer/tutorial-ai-builder?WT.mc_id=aiml-14201-cassieb).

## Conclusion

We went over a fraction of the things that you can do with Form Recognizer so don't let the learning stop here! Check out the below highlights of new Form Recongizer features that were just announced and the additional doc links to dive deeper into what we did here.

### Additional resources
[New Form Recognizer Features](https://azure.microsoft.com/blog/new-features-for-form-recognizer-now-available/?WT.mc_id=aiml-0000-cassieb#:~:text=New features for Form Recognizer now available. Neta,tables from documents to accelerate their business processes.)

[What is Form Recognizer?](https://docs.microsoft.com/azure/cognitive-services/form-recognizer/overview?WT.mc_id=aiml-14201-cassieb)

[Quickstart: Use the Form Recognizer client library or REST API](https://docs.microsoft.com/azure/cognitive-services/form-recognizer/quickstarts/client-library?tabs=preview,v2-1&pivots=programming-language-python%3FWT.mc_id%3Daiml-14201-cassieb&WT.mc_id=aiml-0000-cassieb)

[Tutorial: Create a form-processing app with AI Builder](https://docs.microsoft.com/en-us/azure/cognitive-services/form-recognizer/tutorial-ai-builder?WT.mc_id=aiml-14201-cassieb)