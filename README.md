# Inconnect API Gateway Documentation & Examples

## Services

**public**

* query product information of insurance products in requested language
* query price information and price comparisons
* query doctor lists (health insurance)
* query tariff types and options
* query contact information assets (ZIP codes)
* query reference assets (list of insurers, ids and category mappings)
* query session-ids and tokens needed for submitting leads to the Insurando CRM
* submit customers (single leads, households with multiple persons) as json data 

**restricted/internal**

* customer categorization and blacklists
* triage and routing settings

## Endpoint base URL

* https://api.insurando.ch/v1          (live)
* https://api-tst.insurando.ch/v1      (test)

## Interactive API documentation (Swagger)

* https://api-tst.insurando.ch/v1/docs

## API architecture

* /utils ```create session-ids and tokens```
* /products ```product information```
* /sales ```submit opportunities & household data```
* /assets ```mapping and option lists```


## Concepts

**Opportunity**

when submitting customers to inconnect (sales) you must create an opportunity json object. An opportunity can contain one person or multiple persons (family/household). The Opportunity key holds information about the type and source of the opportunity.

```
{
    "Customers" : []
    "Opportunity": {
        "OpportunityType": "Krankenversicherung",
        "Source": "insurando.ch/krankenkasse"
}
```

**Customer**

Since there can be multiple persons within an opportunity, the first customer in an opportunity is marked as ```{"MainContact" : true}```

```
{
  "Customers": [
    {
      "CustomerId": "hash12345",
      "MainContact": true,
      "ContactInfo": {
        "FirstName": "John",
        "LastName": "Smith",
        "Email": "john.smith@gmail.com"
      }
    }
  ],
  "Opportunity": {
    "OpportunityType": "Krankenversicherung",
    "Source": "insurando.ch/krankenkasse"
  }
}
```

**Session-Id**

before making requests you must create a session-id with GET ```./utils/sessionid``` which you will provide in the Header for future requests.

```diff 
! If you submit multiple opportunities, you must create a new session-id for each Opportunity!
```

**Header parameters**

always provide parameters in the ```Header``` of the request.
the sesion-id is retrieved from GET https://api-tst.insurando.ch/v1/utils/sessionid
```ll
{
    "accept-language: de-CH",
    "x-session-id: "abc",
    "x-google-id": "GA...."
}
```

```diff 
! Currently only de-CH is content-complete! 
```

### Categories

Insurando uses a 2-level categorization schema. 

```diff 
! The category labels must be provided in German to be compatible with the CRM system
```

* Type (high level)
* Category (sub-category)

```
{
    "ProductInsuranceType": "Krankenversicherung",
    "ProductCategory": "Grundversicherung"
}
```

find the list of valid categories https://api-tst.insurando.ch/v1/assets/optionlist/categories

## Sales endpoints

There are two types of endpoints to submit Opportunities:

* ```/sales/contactform``` : generic endpoint for all insurance types, always routed to Insurando CRM
* ```/sales/<specific>``` : endpoints for specific insurance types that can be routed directly to insurance companies APIs

## Example workflow for contest ("Wettbewerb)

**