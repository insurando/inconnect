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
    "x-google-id": "GA1.2.1562643405.1595230641"
}
```

```diff 
! Currently only de-CH is content-complete! 
```

## Categories

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

## Example: basic health insurance with supplementary needs

1. retrieve session-id

```
curl -X GET "https://api-tst.insurando.ch/v1/utils/sessionid"
```

2. provide your frontend a list of insurance companies

```
curl -X GET "https://api-tst.insurando.ch/v1/products/health/basic/insurer" -H  "accept-language: de-CH" -H  "x-session-id: hash12345" -H  "x-google-id: GA.12345"
```

3. find the valid zip code and community names:

```
curl -X GET "https://api-tst.insurando.ch/v1/products/health/basic/zipmunicipalitycity/8057" -H  "accept-language: de-CH" -H  "x-session-id: hash12345" -H  "x-google-id: ga12345"
```

```
[
  {
    "ZipMunicipalityCity": "8057 - Zürich - Zürich",
    "RegionId": "PR-REG CH1",
    "CantonId": "ZH",
    "CommunityNumber": 261
  }
]
```

4. find the available tariff options for that premium region...use **RegionId** and **CantonId** from step 3, user birth-year and **InsurerId** from step 2 to make the POST request

```
curl -X POST "https://api-tst.insurando.ch/v1/products/health/basic/tariff/options" -H  "accept-language: de-CH" -H  "x-session-id: hash12345" -H  "x-google-id: ga12345" -H  "Content-Type: application/json" -d 
"{
    \"InsurerId\":\"8\",
    \"CantonId\":\"ZH\",
    \"RegionId\":\"PR-REG CH1\",
    \"Birthday\":1990
}"
```

```
{
  "TariffType": [
    {
      "Label": "TAR-BASE",
      "Name": "Freie Arztwahl"
    },
    {
      "Label": "TAR-DIV",
      "Name": "Telmed / Apotheke"
    },
    {
      "Label": "TAR-HMO",
      "Name": "HMO"
    },
    {
      "Label": "TAR-HAM",
      "Name": "Hausarzt"
    }
  ],
  "Franchise": [
    "300",
    "500",
    "1000",
    "1500",
    "2000",
    "2500"
  ]
}
```

5. most health insurer require residency permits:

```
curl -X GET "https://api-tst.insurando.ch/v1/products/health/basic/countrycodespermits" -H  "accept-language: de-CH" -H  "x-session-id: hash12345" -H  "x-google-id: ga12345"
```

```
{
"CountryCodes":
    {
      "Code": "ZW",
      "Name": "Zimbabwe"
    },
    {
      "Code": "CY",
      "Name": "Zypern"
    }
  ],
  "Permits": [
    {
      "Code": "B",
      "Name": "Ausweis B (Aufenthaltsbewilligung)"
    },
    {
      "Code": "C",
      "Name": "Ausweis C (Niederlassungsbewilligung)"
    }
  ]
}
```

6. have the user select additional needs:

```
curl -X GET "https://api-tst.insurando.ch/v1/products/health/supplementary/needslist" -H  "accept-language: de-CH" -H  "x-session-id: hash12345" -H  "x-google-id: ga12345" -H  
```

```
...
      {
        "NeedsId": 6,
        "Need": "Alternative Heilmethoden",
        "NeedDesc": ""
      },
      {
        "NeedsId": 7,
        "Need": "Fitness",
        "NeedDesc": ""
      }
...
```

7. generate the final Opportunity payload

* provide contact information of the main contact
* we will use the /sales/contactform endpoint to submit the Opportunity
* optional:
    * provide the current product, if the user provided it
    * provide the new product "Grundversicherung"
    * provide the new product or need "Zusatzversicherung"

```diff 
! In case you provide products, you can leave certain product keys blank with empty "", in case the API demands it
```

```diff 
! for Zusatzversicherung use ProductName=Needs and ProductOptions=Need1,Need2
```

```
curl -X POST "https://api-tst.insurando.ch/v1/sales/contactform/submitlead" -H  "accept-language: de-CH" -H  "x-session-id: hash12345" -H  "x-google-id: ga12345" -H  "Content-Type: application/json" -d
```

```
{
  "Customers": [
    {
      "CustomerId": "hash12345",
      "MainContact": true,
      "ContactInfo": {
        "FirstName": "John",
        "LastName": "Smith",
        "Gender": "male",
        "BirthDate": "1990-01-31",
        "PostCode": "8600",
        "Canton": "ZH",
        "CommunityName": "Dübendorf",
        "CommunityNumber": "123",
        "AdressStreet": "teststrasse",
        "AdressNumber": "12b",
        "Phone": "+41123123123",
        "Email": "john.smith@gmail.com",
        "Language": "de",
        "CountryIso": "CH",
        "ResidencePermit": "B",
        "EmailOptIn": true
      },
      "Products": [
        {
          "ProductInsuranceType": "Krankenversicherung",
          "ProductCategory": "Grundversicherung",
          "ProductType": "current",
          "ProductInsuranceName": "CSS",
          "ProductInsurerId": "8",
          "ProductAccidentCoverage": false,
          "ProductFranchise": "300",
          "ProductTariffType": "Telmed",
          "ProductVariant": "",
          "ProductOptions": ""
        },
        {
          "ProductInsuranceType": "Krankenversicherung",
          "ProductCategory": "Grundversicherung",
          "ProductType": "new",
          "ProductInsuranceName": "CSS",
          "ProductInsurerId": "8",
          "ProductName": "Basisversicherung",
          "ProductAccidentCoverage": true,
          "ProductFranchise": "2500",
          "ProductTariffType": "Freie Arztwahl",
          "ProductVariant": "",
          "ProductOptions": ""

        },
        {
          "ProductInsuranceType": "Krankenversicherung",
          "ProductCategory": "Zusatzversicherung",
          "ProductType": "new",
          "ProductName": "Needs",
          "ProductVariant": "",
          "ProductOptions": "Alternative Heilmethoden,Fitness"
        }
      ]
    }
  ],
  "Opportunity": {
    "OpportunityType": "Krankenversicherung",
    "Source": "website.ch/krankenkasse",
    "OpportunityComment": "Der Kunde ist sich bei der Zahnversicherung unsicher"
  }
}
```

## Example: car insurance

```
{
  "Customers": [
    {
      "CustomerId": "hash12345",
      "MainContact": true,
      "ContactInfo": {
        "FirstName": "John",
        "LastName": "Smith",
        "Gender": "male",
        "BirthDate": "1990-01-31",
        "PostCode": "8600",
        "Canton": "ZH",
        "CommunityName": "Dübendorf",
        "CommunityNumber": "123",
        "AdressStreet": "teststrasse",
        "AdressNumber": "12b",
        "Phone": "+41123123123",
        "Email": "john.smith@gmail.com",
        "Language": "de",
        "CountryIso": "CH",
        "ResidencePermit": "B",
        "EmailOptIn": true
      },
      "Products": [
        {
          "ProductInsuranceType": "Gegenstandsversicherung",
          "ProductCategory": "Autoversicherung",
          "ProductType": "new",
          "ProductName": "Autoversicherung Anfrage",
          "ProductVariant": "",
          "ProductOptions": ""
        }
      ]
    }
  ],
  "Opportunity": {
    "OpportunityType": "Gegenstandsversicherung",
    "Source": "website.ch/auto"
  }
}
```

## Example: Contest (Wettbewerb)

Special use case: website visitors may participate in contests where they provide contact information and also evaluate their current health insurance (Umfrage) or other services. This information is mapped to product configurations

* for OpportunityType "Wettbewerb", a product must be created accordingly
* in this case the product must be created with ProductCategory "Wettbewerb" with the context ProductInsuranceType="Krankenversicherung"
* "Wettbewerb" and "Umfrage" can be combined as well, "Wettbewerb" is the leading category in this case
* for collecting survey information with product entries the following key-value convention applies:
    * ProductName / Title of the survey
    * ProductVariant / Key
    * ProductOptions / Value

```
! see the product configuration for "Wettbewerb" combined with "Umfrage"
```

```
{
  "Customers": [
    {
      "CustomerId": "hash12345",
      "MainContact": true,
      "ContactInfo": {
        "FirstName": "John",
        "LastName": "Smith",
        "Gender": "male",
        "BirthDate": "1990-01-31",
        "PostCode": "8600",
        "Canton": "ZH",
        "CommunityName": "Dübendorf",
        "CommunityNumber": "123",
        "AdressStreet": "teststrasse",
        "AdressNumber": "12b",
        "Phone": "+41123123123",
        "Email": "john.smith@gmail.com",
        "Language": "de",
        "CountryIso": "CH",
        "ResidencePermit": "B",
        "EmailOptIn": true
      },
      "Products": [
        {
          "ProductType": "new",
          "ProductInsuranceType": "Krankenversicherung",
          "ProductCategory": "Wettbewerb",
          "ProductName": "fitness-abo-gutschein",
          "ProductVariant": "",
          "ProductOptions": ""
        },
        {
          "ProductType": "new",
          "ProductInsuranceType": "Krankenversicherung",
          "ProductCategory": "Umfrage",
          "ProductName": "Umfrage: Aktuelle Krankenkasse",
          "ProductVariant": "Aktuelle Krankenkasse",
          "ProductOptions": "Assura"
        },
        {
          "ProductType": "new",
          "ProductInsuranceType": "Krankenversicherung",
          "ProductCategory": "Umfrage",
          "ProductName": "Umfrage: Bewertung",
          "ProductVariant": "Bewertung",
          "ProductOptions": "5"
        }
      ]
    }
  ],
  "Opportunity": {
    "OpportunityType": "Wettbewerb",
    "Source": "website.ch/auto"
  }
}
```
