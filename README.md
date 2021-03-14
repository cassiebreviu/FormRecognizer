# Extract data from PDFs using Form Recognizer Cognitive Service

Form recognizer is a powerful tool to help build a variety of document machine learning solutions.  It is one service however its made up of many prebuilt models that can perform a variety of essential document functions. You can even custom train a model using supervised or unsupervised learning for tasks outside of the scope of the prebuilt models! Read more about all the features of Form Recognizer here. In this example we will be looking at how to use one of the prebuilt models in the Form Recognizer service that can extract the data from a PDF document dataset. Our documents are invoices with common data fields so we are able to use the prebuilt model without using the labeling tool.
After we take a look at how to do this with Python and Azure Form Recognizer, we will take a look at how to do the same process with no code using the Power Platform services: Power Automate and Form Recognizer built into AI Builder.

## Prerequisites for Python
-	Azure Account [Sign up here!]()
-	Anaconda and/or VS Code

## Python and Azure Form Recognizer Service
First lets create the form recognizer cognitive service.
-	Go to portal.azure.com to create the resource or click this [link]()

Now lets create a storage account to store the PDF dataset we will be using in containers. We want 2 containers, one for the `processed` PDFs and one for the `raw` unprocessed PDF.
-	Create an Azure storage account
-	Create containers for training and processing

## Use the  Python SDK and Jupyter notebooks to post to the API

Now that we have our data stored we can connect and create our model using the python sdk for form recognizer. 

- Install the Python SDK:
```python
!pip install azure-ai-formrecognizer --pre
```

- Import the packages

```python
import os
from azure.core.exceptions import ResourceNotFoundError
from azure.ai.formrecognizer import FormRecognizerClient
from azure.ai.formrecognizer import FormTrainingClient
from azure.core.credentials import AzureKeyCredential
```

- Update the endpoint and key with the values
```python 
endpoint = "<your endpoint>"
key = "<your key>"
```

- Create the form recognizer client
```python
form_recognizer_client = FormRecognizerClient(endpoint, AzureKeyCredential(key))
```

- Get url to invoice
```python
invoiceUrl = "https://storageaccount.blob.core.windows.net/train/Invoice.pdf"
poller = form_recognizer_client.begin_recognize_invoices_from_url(invoiceUrl)
invoices = poller.result()
```

- Iterate and print results:
```python 
for idx, invoice in enumerate(invoices):
    print("--------Recognizing invoice #{}--------".format(idx+1))
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

The results should look something like this for the invoice we are using:

![img](/imgs/pythonresult.png)

The prebuilt invoices model worked great for our invoices so we dont need to train a customized Form Recognizer model to improve our results.  But what if we did and what if we didn't know how to program?! You can still leverage all this awesomeness in AI Builder with Power Automate without writing any code. We will take a look at this same example in Power Automate next.

## Use Form Recognizer with AI Build in Power Automate

You can achieve these same results using no code with Form Recognizer in AI Builder with Power Automate.

New Form Recognizer Features

Additional resources
Invoices - Form Recognizer - Azure Cognitive Services | Microsoft Docs
Try your own invoices and samples in the Form Recognizer Sample UI.

