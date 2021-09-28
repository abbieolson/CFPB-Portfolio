## Project 1 - Python-based web scraper

#### *Languages and Tools:*
* Python
  * requests
  * re
  * dateutil
  * datetime
  * pandas
  * csv
  * math
  * os
* Bash
----------
### Makefile:
```bash
venv:
		if [ -d .py ] ; \
		then \
			echo "virtualenv already exists. skipping"; \
		else \
			scl enable rh-python36 "virtualenv .py"; \
			.py/bin/python -m pip install -U pip; \
			.py/bin/python -m pip install -r requirements.txt; \
		fi;

run: venv
	.py/bin/python forecast.py
```

### Functions and Global Variables:
```python3
def main():
    # start timer
    start = datetime.now()

    ########################
    ### global variables ###
    ########################

    user = environ.get("PGUSER", None)
    password =  environ.get("PGPASSWORD", None)
    out_path = environ.get("OUT_PATH", None)
    cfpb_local = environ.get("DOMAIN", None)
    input_csv = 'Investigations_Themes_IPL.csv'
    census_year = 2019

    def wrap_error(func):
        '''Function to keep script alive upon Exceptions'''
        def func_wrapper(*args, **kwargs):
            try:
                return func(*args, **kwargs)
            except:
                pass
        return func_wrapper

    def get_cookies(url, domain):
        '''Function to get csrftoken'''
        r_cookie = requests.get(url,
                    verify=False, 
                    auth=HTTPBasicAuth(user, password))

        cookie_dict = r_cookie.cookies.get_dict(domain=domain)
        return cookie_dict

    def get_api(url, census_year):
        '''Function converts complaint explorer URL to API URL'''
        url = url.replace('&txt=', '&search_term=').replace('&company=', '&sent_to=')

        url_temp = url.split('&')
        sep = '&'

        issues = []
        search = []
        sent_to = []
        product = []
        entity = []

        for i in url_temp:
            if 'issue=' in i:
                issues.append(i)
            elif 'search_term=' in i:
                search.append(i)
            elif 'sent_to=' in i:
                sent_to.append(i)
            elif 'product=' in i:
                product.append(i)
            elif 'entity_name=' in i:
                entity.append(i)
                
        par1 = [f'census_year={census_year}',
                    'date_received_max=2011-07-01',
                    'date_received_min=2011-07-01',
                    'field=what_happened',
                    'frm=0',
                    'index_name=complaint-crdb-prod']
        par2 = ['no_aggs=true']
        par3 = ['size=1000',
                'sort=created_date_desc']

        # the api url is very specific, hence this whacky addition problem            
        par = par1 + issues + par2 + product + search + sent_to + entity + par3
        link = 'https://complaints.data.cfpb.local/api/v2/complaints?' + sep.join(par)
        link = link.replace('(', '%28').replace(')', '%29').replace("'", '%27').replace('*', '%2A').replace(',', '%2C')
        return link

    def get_page_cts(url):
        '''Function to get page counts (large API requests have a size limit per page of either 100 or 1000)'''
        r = requests.get(url, 
                        cookies=cookie_dict,
                        headers=headers,
                        verify=False,
                        stream=True,
                        auth=HTTPBasicAuth(user, password)).json()

        total = int(r['hits']['total'])
        num_pages = math.ceil(total / 1000)
        return num_pages
        
    @wrap_error
    def get_ids(url, name, ipl, count, usr, pswd):
        '''Function to write the theme, IPL, and casenumber to CSV'''
        count = 0
        for i in range(0, get_page_cts(url) + 1):
            if i == 0:
                pass
            else:
                new = url.split('frm=0', 1)[0] + 'frm=' + str(count) + url.split('frm=0', 1)[1]
                count += 1000

                r = requests.get(new, 
                    cookies=cookie_dict,
                    headers=headers,
                    verify=False,
                    stream=True,
                    auth=HTTPBasicAuth(usr, pswd)).json()

                hits = r['hits']['hits']

                for i in hits:
                    line1 = (name, ipl, i['_id'])
                    writer1.writerow(line1)

                    if i['_source']['is_socrata_published'] == 'Yes':
                        line2 = (name, ipl, i['_id'])
                        writer2.writerow(line2)

    # get csrf token
    token_pd = pd.read_csv(input_csv, nrows=1, usecols=['link'])
    token_pd = token_pd.iloc[0]['link']

    # dict with csrf token and session id
    cookie_dict = get_cookies(token_pd, cfpb_local)
    csrf_token = cookie_dict['csrftoken']

    # headers from api (see "how to manually retrieve api urls")
    headers = {
        'Host': cfpb_local,
        'Connection': 'keep-alive',
        'Accept': 'application/json, text/plain, */*',
        "sec-ch-ua": "\" Not;A Brand\";v=\"99\", \"Google Chrome\";v=\"91\", \"Chromium\";v=\"91\"",
        "sec-ch-ua-mobile": "?0",
        'X-CSRFToken': csrf_token,
        'User-Agent': 'User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.111 Safari/537.36',
        'Sec-Fetch-Site': 'same-origin',
        'Sec-Fetch-Mode': 'cors',
        'Sec-Fetch-Dest': 'empty',
        'Referer': 'https://complaints.data.cfpb.local/',
        'Accept-Encoding': 'gzip, deflate, br',
        'Accept-Language': 'en-US,en;q=0.9'
    }  
```

### Main Script:
```python3
    # opens two csvs, one with all ipls, and the other with only those published in ccdb
    with open(out_path + 'all_themes.csv', 'w', newline='') as f1, open(out_path + 'ccdb_themes.csv', 'w', newline='') as f2:
        writer1 = csv.writer(f1)
        writer2 = csv.writer(f2)

        # opens the files and writes out the headers
        writer1.writerow(out_fields)
        writer2.writerow(out_fields)

        # opens the input csv from the investigations team (edited)
        with open(input_csv, 'r') as f:
            reader = csv.DictReader(f, fieldnames=in_fields)
            next(reader)

            # gets the number of rows in the csv file
            for index, row in enumerate(reader):
                index_ct = index + 1
                # converts the input url from the csv file to the api format
                row['link'] = get_api(row['link'], census_year)
                # makes minor adjustments to the credit reporting urls
                # finds the dates in the url string (format: yyyy-mm-dd)
                result = re.findall('\d{4}-\d{2}-\d{2}', row['link'])

                # takes the count from num_weeks and creates another url with a new week
                # increments by week for the length of num_count
                while nums <= (num_months * index_ct):
                    for i, j in enumerate(result):
                        result[i] = (datetime.strptime(j, '%Y-%m-%d') + relativedelta(months =+ 1)).strftime('%Y-%m-%d')
                    nums += 1

                    # splits the url so the dates can be edited
                    url_temp = row['link'].split('&')
                    # gets the date from the while loop (j) and adds it to "date_received_max=" for each url    
                    for k, l in enumerate(url_temp):
                        if 'date_received_max=' in l:
                            this_month = datetime.strptime(j, '%Y-%m-%d')
                            next_month = datetime.strptime(j, '%Y-%m-%d') + relativedelta(months =+ 1)
                            total_days = (next_month - this_month).days
                            url_temp[k] = 'date_received_max=' + str(datetime.strptime(j, "%Y-%m-%d").replace(day=total_days).date())

                        # uses j to create "date_received_min=", subtracts by week 
                        # makes each week 6 days long rather than 7 because elastic search is date inclusive (don't want duplicates)
                        elif 'date_received_min=' in l:
                            url_temp[k] = 'date_received_min=' + j

                    # joins the split url back together with '&' once the dates have been adjusted
                    url = sep.join(url_temp)
                    # function that actually gets the ids from the urls and writes them to a file
                    get_ids(url, row['name'], row['IPL'], count, user, password)

if __name__ == "__main__":
    main()
##### END
```
