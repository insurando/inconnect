# Inconnect API Gateway Documentation & Examples

### Services

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

### Endpoint base URL

* https://api.insurando.ch/v1          (live)
* https://api-tst.insurando.ch/v1      (test)

### Interactive API documentation (Swagger)

* https://api-tst.insurando.ch/v1/docs

### API architecture

* /utils ```create session-ids and tokens```
* /products ```product information```
* /sales ```submit opportunities & household data```
* /assets ```mapping and option lists```


### Concepts

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

**Session-Id**

before making requests you must create a session-id with GET ```./utils/sessionid``` which you will provide in the Header for future requests

**Header parameters**

always provide parameters in the ```Header``` of the request.
the sesion-id is retrieved from GET ```./utils/sessionid```
```ll
{
    "accept-language: de-CH",
    "x-session-id: "abc",
    "x-google-id": "GA...."
}
```

| WARNING: currently only **de-CH** is content-complete! |
| --- |
