---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Querying the EITI API"
#subtitle: "A simple python program to automate the SQL query building for Wharton's WRDS"
summary: ""
authors: []
tags: []
categories: []
date: 2022-03-24T17:30:21+11:00
lastmod: 2022-03-24T17:30:21+11:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---


Class to interact with and query the [EITI API](https://api.eiti.org/#eiti---api-documentation).

```python
# Load Libraries
import os
import requests # So we can query a web API
from urllib.parse import urlencode # So we can manipulate the API requests and define the search
import pandas as pd
import json
import datetime
from termcolor import colored, cprint # to print errors in red!
import numpy as np
```


```python
class EITIstream():
    """
    Container to manage EITI API
    """

    def __init__(self, version, section, headers, params=None):
        self.headers = headers
        self.key = version
        self.section = section
        self.url = 'https://eiti.org/api/' + version + "/" + section
        if params:
            self.response = dict(requests.get(self.url, params=params, headers = headers).json())
        else:
            self.response = dict(requests.get(self.url, params={}, headers = headers).json())
        self.current = 0


    """def hasNext(self):
        if (self.current >= self.response['count']):
            return(False)
        else:
            return(True)"""

    def hasNext(self):
        """
        Helper funtion to check if there exists a next page
        """
        if "next" in list(self.response.keys()):
            return(True)
        else:
            return(False)

    def nextPage(self):
        """
        Helper funtion that graps the next page
        """
        nextPage = self.response['next']['href']
        self.response = dict(requests.get(nextPage).json())


    def get_organisations(self):
        """
        Method to collect information on all organizations later to be matched with
        revenue data.
        ----
        Return: List of Dictionaries
        """  
        while self.hasNext(): # while there exists a new page
            data = self.response['data'] # retrieve data
            self.current += len(self.response['data']) # update count by number of entries retrieved in this step
            self.nextPage() # update url
            return(data)
        else:
            data = self.response['data'] # retrieve data
            self.current += len(self.response['data']) # update count by number of entries retrieved in this step
            return(data)

    def get_countries(self):
        """
        Method to collect information on all countries in the EITI dataset
        ----
        Return: DataFrame
        """  
        data = self.response['data'] # retrieve data
        count = self.response['count'] # number of reports

        if count == len(data): # sanity check

            countries = [] # initalize country list

            for country in data: # loop through countries

                if country['reports']:
                    # if there exist reports write report years in ascending order as comma sperated string
                    years = list(country['reports'].keys())
                    years.sort()
                    report_years = ",".join(years)
                else:
                    report_years = None

                info = {'country_id': country['id'],
                        'country': country['label'],
                        'iso2': country['iso2'],
                        'iso3': country['iso3'],
                        'report_years': report_years,
                        'latest_validation_date': country['latest_validation_date'],
                        #'latest_validation_url': country['latest_validation_url']
                       }

                countries.append(info) # append info for country

            return pd.DataFrame(countries)
        else:
            cprint("Error: the response count does not equal the number of data reports retrieved", "red")

    def get_JoinLeaveDates(self):

        data = self.response['data'] # retrieve data
        count = self.response['count'] # number of codes

        if version == "v2.0": # sanity check as data only retrievable if version 2

            if count == len(data): # sanity check
                info = [] # empty list to save dictionaries
                for country in data:
                    dic = {"id_v2": country['id'],
                           "country": country['label'],
                           "iso2": country['iso3'],
                           "iso3": country['iso3'],
                           "join_date": country['join_date'],
                           "leave_date": country['leave_date'],
                           "latest_validation_date": country['latest_validation_date']
                          }
                    info.append(dic)

                return pd.DataFrame(info)

            else:
                cprint("Error: The response count does not equal the number of data reports retrieved", "red")
        else:   
            cprint("Error: Requested Data only retrievable in 'v2.0'. Check Version!", "red")

    def get_gfsCodes(self):
        """
        Method to collect information on all GFS Codes/Descriptions in the EITI dataset
        ----
        Return: DataFrame
        """  
        data = self.response['data'] # retrieve data
        count = self.response['count'] # number of codes

        if count == len(data): # sanity check

            codes = [] # initalize

            for code in data: # loop through codes

                info = {'gfs_code_id': code['id'],
                        'gfs_code': code['code'],
                        'gfs_description': code['label'],
                        'gfs_parent': code['parent']
                        }

                codes.append(info) # append info for code

            return pd.DataFrame(codes)
        else:
            cprint("Error: the response count does not equal the number of data reports retrieved", "red")


    def get_revenues(self):
        """
        Method to collect information on revenues paid by companies.
        ----
        Return: DataFrame
        """  

        data = self.response['data'] # retrieve data
        count = self.response['count'] # number of reports

        if count == len(data): # sanity check

            if data: # if there exists data

                reports = [] # initalize reports list

                for report in data: # loop through reports

                    print(f"Working on {report['label']}")

                    # save contextual info as a dictionary
                    info = {'report_id': report['id'],
                            'report_label': report['label'],
                            'country_id': report['country']['id'],
                            'country': report['country']['label'],
                            'government_entities_nr': report['government_entities_nr'], # number of goverment organizations\
                            'company_entities_nr': report['company_entities_nr'], # number of company organizations
                            'publication_date': report['publication_date_EITI_report'], # format of date???
                            'year_start': report['year_start'],
                            'year_end': report['year_end']
                           }

                    # save renues data as DataFrame
                    rev_gov = report['revenue_government']
                    rev_comp = report['revenue_company']
                    revenues = []
                    for rev in [rev_gov, rev_comp]: # to accomodate cases when revenue data is empty
                        if rev:
                            revenues.extend(rev)

                    if revenues: # if revenue info exists
                        df = pd.DataFrame(revenues)
                        # add contextual info to DataFrame
                        for key in info.keys():
                            loc = df.shape[1] # position
                            df.insert(loc, column = key, value = info[key])
                    else:
                        df = None

                    # append report
                    reports.append(df)


                # concat DataFrames for country, else return None
                return pd.concat(reports, ignore_index=True)

            else: # if no data exists
                return None
        else:
            cprint("Error: the response count does not equal the number of data reports retrieved", "red")
```
