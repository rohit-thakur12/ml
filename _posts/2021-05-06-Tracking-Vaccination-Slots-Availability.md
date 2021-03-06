---
title: "Tracking Availability of Vaccination Slots using COWIN's API"
toc: false
comments: true
layout: post
categories: [actions, markdown]
image:
author: Rohit Thakur
---

Everyone wants to be vaccinated against covid-19. The only problem being not enough vaccination slots available. With the population being in billions and the daily vaccination doses being barely 50, it almost seems impossible to get a slot.

This is a guide on how to build an application that can automatically monitor COWIN for vaccination slots. 

We will use COWIN's api to first get all the centres in the specified pincode, extract the information required from that response, check for the availability of slots and send a message to our phone or an email letting us know if the slots are available.

## Getting the data

Getting the data is easy as it has been open for everyone to use thourgh the government's [API Setu](https://apisetu.gov.in/public/api/cowin) program. 

We have options in how we wants to filter the data. For our app we will use the one that says "calenderByPin". 

## Setting up twilio for SMS

[Twilio](https://www.twilio.com/) is a service that allows us to send text messages for free to anyone and anywhere. We will use this to send a text message to our phone whenever a slot is available. 

You can sign up for a free account and once your account is up and running, look for SID, Auth Token and your twilio phone number. You will find these on your dashboard. 

## Setting up an email notification service

Even though we send text messages in his app, you can create an email alert system if that suits you. 

You can do this using "smtplib" which is a library in python that helps you send email alerts. 

## Building the application

### First we start with importing the necessary libraries.

```python
import requests
import json
from datetime import datetime
from twilio.rest import Client
```
We use "requests" to send requests to the API. We are also using a json file to store all the configurations we need so we can change it whenever we want without breaking the app. Next we import "datetime" which is just so we can get the current date and last we import "twilio" which enables us to send messages. 

NOTE: We have used json here but depending on the sensitivity of the data, you can either use environment variables or a ".ini" file and parse it using python's "configparser".

### Setting up logging.

```python
import logging
logging.basicConfig(level=logging.INFO, filename="vaccine.log",format='%(asctime)s - %(message)s')  

```

We set up logging to keep a check on our app. 

### Load the date and config file.

```python
date = datetime.today().strftime('%d-%m-%Y')

with open("config.json", "r", encoding="utf-8") as file:
      data = json.load(file) 
```

We load the date and the config file. The config file for now looks like this: 

```json
    {
    "pincode":[410501, 410505],
    "age": 18,
    "twilio-sid": "##",
    "twilio-auth": "##",
    "twilio-number": "##",
    "myNumber": "##"
    }
```
You can enter your details in the file. You can add or remove pincodes. Currently the only two age groups that can get vaccinated is people between 18-44 and people above 45 years of age. To get details about vaccination for people above the age of 45, change the "age" value to 45.

### Parsing twilio credentials.

```python
    accountSid = data['twilio-sid']
    authToken = data['twilio-auth']
    client = Client(accountSid, authToken)
    myTwilioNumber = data['twilio-number'] 
    destCellPhone = data['myNumber']

 ```

 We parse the information that we need to use twilio's API. 

### Builing the funtion to monitor the API

```python
 def main():
    for pincode in data['pincode']:
        _URL = f"https://cdn-api.co-vin.in/api/v2/appointment/sessions/public/calendarByPin?pincode={pincode}&date={date}"
        response = requests.get(_URL)
        if response.status_code != 200:
            logging.info(f"Error with code {response.status_code}")
        else:
            scrape = response.json()
            for i in scrape['centers']:
                sess = i['sessions']
                for sessions in sess:
                    if sessions['available_capacity'] > 0 and sessions['min_age_limit'] == data['age']:
                        messageBody = f"Vaccine available at {i['name']} -  {i['address']} - {i['pincode']} on {sessions['date']} and the variant is {sessions['vaccine']}"
                        client.api.account.messages.create(to= destCellPhone, from_=myTwilioNumber, body=messageBody) 
                    else:
                        logging.info(f"Vaccine not available at {i['name']} - {i['pincode']}")
```

This is the whole function that is used to monitor the slots. We loop through all the pincodes in we want and look for availability. If the slots are available, we will get a text message. 

### Scheduling the script to run automatically. 

Now that we have everything that we need, we can schedule the script to run automatically with the frequency we need. To do this, we can use Task Scheduler on Windows or CRON on linux/MAC. 

## Ending thoughts

Being a government website, it is unavailable ususally in the morning so dont worry if you get a 403 error. 

Unfortunately we cannot schedule appointments directly using the API but you can contact the Ministry of Health if you plan on doing that like it says on the website. I hope this helps. 

GITHUB: https://github.com/rohit-thakur12/VaccineTrackerIndia
