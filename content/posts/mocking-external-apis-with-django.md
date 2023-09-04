+++ 
draft = false
date = 2021-02-24
title = "Mocking External APIs in Django"
description = ""
slug = "mocking-external-apis-in-django"
authors = ["Ashwin Easo Zachariah"]
tags = []
categories = []
externalLink = ""
series = []
+++

---

_Most articles and SO posts that you read suggest mocking the API calls using the Mock and patch features in the unitest library while running your tests. But, what if you are trying to manually test the flow on your local machine or any other controlled environment?_

While building a backend service for a web application, its common to come across a scenario where you have to integrate with an external service and use its APIs.

## Setting Up a Scenario
Consider a case where we are building an e-commerce application. Our application is designed to handle the marketplace and we have to integrate with a third-party service like FedEx to handle the shipping and logistics.

Assume that FedEx service offers us 2 APIs
1. Create Shipment API: To create a new shipment order.
2. Shipment Status API: To check the status of the shipment.

For the sake of the article, I will be assuming that we are using an adapter pattern to build the interface with the third party service. And so, we end up with a service class with two methods as shown below which calls the actual APIs of the FedEx service.

```python
class FedexService:
    """Service class for interfacing with FedEx Service."""
    def create_shipment(self, shipment_details: dict):
        """Create the shipment."""
        # Call the API to create shipment
    
    def check_status(self, shipment_id: str):
        """Check the status of the shipment by the id."""
        # Call the API to check the status
```

However, for testing purposes, we would want to avoid calling the FedEx service APIs, and instead call another service that would implement the same methods, but would return a pre-determined/ controlled response.

We can implement the mocked service as shown

```python
class MockShippingService:
    """Mock service class for shipping."""
    def create_shipment(self, shipment_details: dict):
        """Create the shipment."""
        return {
            'shipping_id': str(uuid.uuid4()),
            'created_at': datetime.datetime.isoformat(),
            'charges': '100.10'
        }
    
    def check_status(self, shipment_id: str):
        """Check the status of the shipment by the id."""
        return {
            'status': 'delivered',
            'updated_at': datetime.now().isoformat()
        }
```

## Switching the services seamlessly
Once we have the actual and the mocked service classes implemented, the only thing that remains is the switching between those classes as and when required. The trick to that lies in implementing a multiplexer:

```python
@apiplexer
class ShippingService:

    api_settings = 'SHIPPING_SERVICE'
This class acts as the interface which is used by the rest of the application to call the shipping service. For example:
def create_shipment(self, order_details: dict):
    """Create the shipment for the order."""
    # Do something to create the shipment_details
    shipment = ShippingService().create_shipment(shipment_details)
    # Do something to handle the shipment
```

The apiplexer decorator is the key to switching between the service classes. The decorator can be implemented as:

```python
from django.conf import settings
from django.utils.module_loading import import_string
def apiplexer(cls):
    """Load the classes based on their names."""
    return import_string(getattr(settings, cls.api_settings))
```

So how does the actual switching work? For that, we specify the classes to use in our settings files.  
In `settings/local.py` we insert

```
SHIPPING_SERVICE = 'shipping.service.MockShippingService'
```

and in `settings/production.py` we add

```
SHIPPING_SERVICE = 'shipping.service.FedexService'
```

When the Django app is loaded, the appropriate shipping service class to be used is imported based on the value of the `SHIPPING_SERVICE` provided in the settings for that environment.


## Extend as required
The MockShippingService can further be extended to handle a lot of different cases based on the values that are passed to its methods. It can return prefixed values; or fetch and return values from the database, or maybe even call a mock API server.
The point of using this method is to have a way to interface with a third-party service without having to actually communicate with it. This makes it easier to test manually in a local or staging environment with a minimal amount of external dependencies and resources involved.
