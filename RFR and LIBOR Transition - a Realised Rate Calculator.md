# Risk-Free Rates and the LIBOR Transition - a Realised Rate Calculator

As you will be aware, the FCA is phasing out the LIBOR rate for various reasons and it is expected to  cease in the near future.
Market participants are expected to transition away from LIBOR to adopt alternatives - Risk-Free Rates (RFR).

In the UK, for example. a reformed version of the Sterling Overnight Index (SONIA) is the recommended replacement as the preferred RFR for sterling Markets after the end of 2021. Other jurisdictions have alternative rates being proposed or developed - such as Secured Overnight Rate (SOFR) in the USA.

Refinitiv offers considerable resources and tools to help in the migration plan such a dedicated IBOR Transition App in our Eikon / Workspace desktop offerings, as well as other initiatives around Term Reference Rates, IBOR fallback language and Derived analytics.

From a developer's perspective the Refinitiv Data Platform APIs can also help in the migration process.

In this two-part article, I will explore two ways in which the RDP APIS and the higher level RDP Library can be used by developers to assist in this migration effort. 

Firstly, we will look at how you can use the Financial Contracts API offered by our Instrument Pricing Analytics to calculate Realised Risk-Free Rates. I will be focusing on SONIA - no doubt a similar technique could be applied to calculate other rates. 

In the 2nd part, I will look at how you could plot Zero-Coupon Curves based on an RFR rather than LIBOR.

## Calculating Risk-Free Rates

So, how can the Refinitiv Data Platform help you migrate to RFR?

The Data Platform offers a Financial Contracts API which we can use to create an Interest Rate Swap instrument. For example, I could define an IR Swap instrument to calculate the Accrued of its float leg - and use it to replicate the NatWest Reference rate specification (one of the earlier SONIA based benchmarks).

To demonstrate this, I am going to implement a basic Realised Rate calculator using the Financial Contracts API.

The Financial Contracts API can be accessed using <a href="https://developers.refinitiv.com/refinitiv-data-platform/refinitiv-data-platform-apis" target="_blank">our REST API interface</a> at <a href="https://api.refinitiv.com/data/quantitative-analytics/v1/financial-contracts" target="_blank">https://api.refinitiv.com/data/quantitative-analytics/v1/financial-contracts</a> 

However, to make things as simple as possible,  I will be using the Python version of our <a href="https://developers.refinitiv.com/refinitiv-data-platform/refinitiv-data-platform-libraries" target="_blank">Refinitiv Data Platform Library</a> - which provides an ease of use wrapper around the REST APIs.

I have previously written some articles around the RDP Library so I won't go into much detail on how to use the library itself. If you have not already done so, I recommend you <a href="https://developers.refinitiv.com/article/discover-our-refinitiv-data-platform-library-part-1" target="_blank">read the articles</a> for an overview.

### Financial Contracts API

The Financial Contracts API is an incredibly flexible one - which you can explore in more detail on the <a href="https://apidocs.refinitiv.com/Apps/ApiDocs#/details/L2RhdGEvcXVhbnRpdGF0aXZlLWFuYWx5dGljcy92MQ==/L2ZpbmFuY2lhbC1jb250cmFjdHM=/POST/PLAYGROUND" target="_blank">API Playground</a> or via the <a href="https://developers.refinitiv.com/elektron-data-platform/elektron-data-platform-apis/docs?content=61175&type=documentation_item" target="_blank">RDP API Documentation pages</a>.

In addition to Interest Rate Swaps, the Financial Contracts API supports FX Spots, FX Forwards, FX Swaps, Credit Default Swaps, Bonds, Equity options as well as several other types of financial contracts.

In order to use the **Financial Contracts API** via the RDP Library, I need to create a session and an Endpoint object referencing the Financial Contracts API:
```python
import refinitiv.dataplatform as rdp
session = rdp.DesktopSession(app_key)
session.open()

endpoint = rdp.Endpoint(session,  
    'https://api.refinitiv.com/data/quantitative-analytics/v1/financial-contracts')
```

I can now use the Endpoint object to submit a IR Swap definition to the platform and receive the response data back from the Platform. So, I will define a Swap request to provide an Accrued Percentage value replicating the NatWest Reference Rate.

To submit my IR Swap Request I need to provide the following inputs:
* Start date of the Interest Rate Swap matching the start date of the realised rate period  
* Maturity date of the Interest Rate Swap matching the end date of the realised rate period  
* Amount used to compute the interest paid over the realised rate period
* Currency of the amount  
* Spread in Basis point that is added to the realised rate  
* Day count basis used to compute each daily year fraction  
* Name of the reference index used to compute the realised rate (e.g. SONIA, SOFR)  
* Look back period i.e. number of days between the daily interest period and the reference index publication( where 0 means the interest period uses the fixing of the previous day)

#### Define my IR Swap

Below is my ```create_swap``` function to generate the JSON for my Swap definition - a straightforward '*fixed for floating*' swap - utilising the above inputs:

```python
def create_swap(start_date, end_date, amount, spread_bp, day_count, currency, index_name, look_back):
 swap_definition = {
      "instrumentType":"Swap", 
      "instrumentDefinition": {
          "instrumentTag":"Compounding-Rate",
          "startDate":start_date,
          "tenor":"1Y",
          "legs":[
          {
              "legTag": "Fixed",
              "interestType": "Fixed",
              "notionalCcy": currency,
              "notionalAmount": amount,
              "interestCalculationMethod": day_count,
              "direction":"Paid",
              "interestType":"Fixed",
              "interestPaymentFrequency": "Zero",
              "fixedRatePercent":1.0,
          },
          {
              "legTag": "Float",
              "direction":"Received",
              "interestType":"Float",
              "notionalCcy": currency,
              "notionalAmount": amount,
              "spreadBp": spread_bp,
              "interestPaymentFrequency": "Zero",
              "interestCalculationMethod": day_count,
              "indexTenor": "ON",
              "indexName": index_name,
              "indexFixingLag": look_back,
              "indexCompoundingMethod": "Compounded",
          }]
      },
      
      "pricingParameters": {
          "valuationDate": end_date,
      },
   }
return swap_definition
```
Note the two legs for my swap; the fixed leg, followed by the float - using the supplied parameters to generate a JSON payload - which I can then send as part of the request to the API endpoint.

#### Submit my request to the API Endpoint

I use the JSON returned by the above ```create_swap``` function as part of my fuller request payload:

```python
request_body = {

    "fields" : ["InstrumentTag","LegTag","MarketValueInDealCcy","AccruedPercent", "AccruedAmountInDealCcy", "ErrorCode","ErrorMessage"],

    "universe" : [
        create_swap(start_date, end_date, amount, spread_bp, day_count, currency, index_name, look_back)
        ],

    "outputs" : ["Data","Headers"],
}

swap = endpoint.send_request(
    method = rdp.Endpoint.RequestMethod.POST,
    body_parameters = request_body)
```
In the above request, the input is the Swap Definition JSON and the expected output consists of a selection of data fields and headers. I am requesting the headers so I can display the response in a user-friendly dataframe - as well as the ability to access the response fields by name.

```python
headers_name = [h['name'] for h in swap.data.raw['headers']]
df = pd.DataFrame(data=swap.data.raw['data'], columns=headers_name)
display(df)
```
Which should generate something like this:  
![](CreateSwapResponse.png)

From the list of fields I received, I am interested in the AccruedPercent for the Floating rate i.e.:
```python
    return df['AccruedPercent'][1]
```

### Calculate the Year Fraction of the Realised Rate Period

I now have an Accrued Percentage, however, to get our actual **annualised** Realised Rate value - I also need to work out the year fraction based on my start and end dates.

Rather than computing this myself,  I can use another RDP API - the **Quantitative Analytics Dates & Calendar API**. This offers various endpoints such as daily schedules, holidays as well as **count-periods** - to get the period of time between a start date and an end date using one or more calendars.

```python
cp_endpoint = rdp.Endpoint(session, "https://api.refinitiv.com/data/quantitative-analytics-dates-and-calendars/beta1/count-periods")

    cp_request_body=[
        {
        "tag": "my request 1",
        "startDate": start_date,
        "endDate": end_date,
        "DayCountBasis":day_count_basis,
        "periodType": period_type,
        "calendars": [calendars]
        }
    ]

    period_of_time = cp_endpoint.send_request(
        method = rdp.Endpoint.RequestMethod.POST,
        body_parameters = cp_request_body)
        
    df = pd.json_normalize(period_of_time.data.raw)
    display(df)
    
    return period_of_time.data.raw[0]['count']
    
```
I will use the same day count basis as the Swap and GBP Calendar. The response gives me something like:    
![](CountPeriodsResponse.png)
- from which **count** is the value I am interested in.

Using the Accrued Percent and the above Period of Time value, I can now calculate the Realised Rate
```python
    return round(accrued_percent / period_of_time,4)
```

In the accompanying source code I have created a simple UI to make it easier to calculate the Realised Rate with input values of my choice:

![Rate Calculator UI](RateCalculatorUIb.png)

Given the input values of 24/02/2020 to 24/03/2020 for £10,000 GBP (therefore SONIA) on a 365-day count basis, I get a realised rate of **0.4669** - which is the same as the value generated by NatWest Reference rate generator.

### Summary

Just to recap that I calculated our Annualised Risk-Free rate with the Refinitiv Data Platform using the:
* Financial Contract API to submit an IR Swap with user-defined parameters to obtain the Accrued of its float leg
* Quantitative Analytics API to calculate the Year Fraction of the RFR Period 

This was just one example of how the Financial Contracts API can assist you as part of the LIBOR transition or other workflows. As I mentioned at the start, it also provides coverage for various other asset classes such as 
* FX Spots, FX Forwards, FX Swaps
* CDS, Bonds, Bonds Futures
* Options, Term Deposits and so on..

However, the Financial Contracts API is just one of the Instrument Pricing Analytics APIs which also offers Volatility Surfaces and Curves. I have written about Volatility Surfaces in <a href="https://developers.refinitiv.com/article/instrument-pricing-analytics-volatility-surfaces-and-curves" target="_blank">a previous article</a>, so in the 2nd part of this article, I will focus on Curves - specifically Zero-Coupon Curves.

Before we move onto the 2nd part, I am sure you agree that it did not take much effort at all to calculate my Annualised Realised RFR - by using the Refinitiv Data Platform and the Financial Contracts API.









