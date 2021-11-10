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
* PostgreSQL
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
	.py/bin/python get_ipls.py
```

### Functions and Global Variables:
```python3
def main():

    pg_sslmode = environ.get("PGSSLMODE", "require")
    pg_user = environ.get("PGUSER", None)
    pg_password =  environ.get("PGPASSWORD", None)
    pg_host = environ.get("PGHOST", None)
    pg_port = environ.get("PGPORT", 5432)
    pg_database = environ.get("PG_DATABASE", None)
    cfpb_local = environ.get("DOMAIN", None)

    # input csv from the investigtations team (edited somewhat)
    input_csv = 'Investigations_Themes_IPL.csv'

    # census year
    census_year = 2019
    
    def create_db_connection(user_name, user_password, host_name, port_name, db_name, sslmode):
        '''Function to create database connection'''
        connection = None
        connection = psycopg2.connect(user=user_name,
                                    password=user_password,
                                    host=host_name,
                                    port=port_name,
                                    database=db_name,
                                    sslmode=sslmode)
        
        print("PostgreSQL Database connection successful")
        
        # Create a cursor to perform database operations
        cursor = connection.cursor()
        # Executing a SQL query
        cursor.execute("SELECT version();")
        # Fetch result
        return connection

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
                    auth=HTTPBasicAuth(pg_user, pg_password))

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
                    'date_received_max=2011-07-01', ### FOR BULK RUN ###
                    'date_received_min=2011-07-01',
                    # 'date_received_max=' + (datetime.now() - relativedelta(months = 1)).strftime('%Y-%m-%d'), ### FOR SUBSEQUENT RUNS ###
                    # 'date_received_min='+ (datetime.now() - relativedelta(months = 1)).strftime('%Y-%m-%d'),
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
                        auth=HTTPBasicAuth(pg_user, pg_password)).json()

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
                    line1 = [(name, ipl, i['_id'])]
                    cursor.execute(insert_query, line1)
                    connection.commit()

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
    # establish connection
    connection = create_db_connection(pg_user, pg_password, pg_host, pg_port, pg_database, pg_sslmode)
    cursor = connection.cursor()

    # query to insert into database table
    insert_query = """INSERT INTO crdw.themes_ipls(theme, ipl, casenumber) VALUES %s;"""

    # CLEAR TABLE BEFORE INITIAL RUN THEN COMMENT OUT
    delete_query = """DELETE FROM crdw.themes_ipls;"""
    cursor.execute(delete_query)

    # headers and url separator
    in_fields = ['name', 'IPL', 'link']
    sep = '&'

    ### FOR BULK RUN ###
    # calculates the number of months since cfpb's opening day
    d1 = datetime(2011, 6, 1)
    d2 = datetime.now()
    num_months = (d2.year - d1.year) * 12 + (d2.month - d1.month)

    ### FOR SUBSEQUENT RUNS ###
    # d1 = datetime.now()
    # d2 = datetime.now() + relativedelta(months=1)
    # num_months = (d2.year - d1.year) * 12 + (d2.month - d1.month)

    # total rows in the input csv
    index_ct = 0
    # gets the count of the number of months by the total rows in the input csv
    nums = 0
    # used by the get_ids() function
    count = 0

    # opens the input csv from the investigations team (edited)
    with open(input_csv, 'r') as f:
        reader = csv.DictReader(f, fieldnames=in_fields)
        next(reader)

        # gets the number of rows in the csv file
        for index, row in enumerate(reader):
            index_ct = index
            # converts the input url from the csv file to the api format
            row['link'] = get_api(row['link'], census_year)
            # finds the dates in the url string (format: yyyy-mm-dd)
            result = re.findall('\d{4}-\d{2}-\d{2}', row['link'])

            # takes the count from num_months and creates another url with a new month
            # increments by month for the length of nums
            while nums <= (num_months * index_ct):
                for i, j in enumerate(result):
                    result[i] = (datetime.strptime(j, '%Y-%m-%d') + relativedelta(months = 1)).strftime('%Y-%m-%d')
                nums += 1

                # splits the url so the dates can be edited
                url_temp = row['link'].split('&')
                # gets the date from the while loop (j) and adds it to "date_received_max=" for each url    
                for k, l in enumerate(url_temp):
                    if 'date_received_max=' in l:
                        this_month = datetime.strptime(j, '%Y-%m-%d')
                        next_month = datetime.strptime(j, '%Y-%m-%d') + relativedelta(months = 1)
                        total_days = (next_month - this_month).days
                        url_temp[k] = 'date_received_max=' + str(datetime.strptime(j, "%Y-%m-%d").replace(day=total_days).date())

                    # uses j to create "date_received_min=", subtracts by week 
                    # makes each week 6 days long rather than 7 because elastic search is date inclusive (don't want duplicates)
                    elif 'date_received_min=' in l:
                        url_temp[k] = 'date_received_min=' + j

                # joins the split url back together with '&' once the dates have been adjusted
                url = sep.join(url_temp)
                # function that actually gets the ids from the urls and writes them to a file
                get_ids(url, row['name'], row['IPL'], count, pg_user, pg_password)

if __name__ == "__main__":
    main()
##### END
```
### PostgreSQL for Tableau Extract
```sql
With 
complaint as (select r.casenumber, r.matched_company, r.analyticalproduct, r.analyticalissue, r.analyticalsubproduct
from crdw.reporting r
where r.type = 'Mosaic Complaint'
and   (r.investigationdisposition <> 'Duplicate' or r.investigationdisposition is null)
and   (r.reason_close_with_no_action not in ('Duplicate (CFPB Spotted)', 'Duplicate (Company Spotted)') or r.reason_close_with_no_action is null)
),

-- furnishing exclusion
furnishing_exclusion as (
select a from (VALUES 
('REDACT')) s(a)
),

-- furnishing issues
furnishing_issues as (
select a from (VALUES 
('REDACT')) s(a)
)

---------

select casenumber,
       unnest(array_remove(array[mortgage_servicing_c,
                                 mortgage_origination_c,
                                 auto_servicing_c,
                                 auto_origination_c,
                                 title_loan_c,
                                 personal_loan_c,
                                 payday_loan_c,
                                 federal_student_loan_c,
                                 private_student_loan_c,
                                 credit_card_c,
                                 deposit_c,
                                 credit_reporting_3ncrc_c,
                                 credit_reporting_small_crc_c,
                                 debt_collection_c,
                                 money_service_c,
                                 prepaid_card_c,
                                 furnishing_c], null)) as ipl
   
from (

select  c.casenumber,

-- mortgage servicing
case when c.analyticalproduct = 'Mortgage' 
and c.analyticalissue in ('REDACT')
then 'Mortgage servicing' 
end as mortgage_servicing_c,

-- mortgage origination
case when c.analyticalproduct = 'Mortgage' 
and c.analyticalissue in ('REDACT')
then 'Mortgage origination' 
end as mortgage_origination_c,

-- auto servicing
case when c.analyticalproduct = 'Vehicle loan or lease' 
and c.analyticalissue in ('REDACT')
then 'Auto servicing' 
end as auto_servicing_c,

-- auto origination
case when c.analyticalproduct = 'Vehicle loan or lease' 
and c.analyticalissue in ('REDACT')
then 'Auto origination' 
end as auto_origination_c,

-- title loan
case when c.analyticalproduct = 'Title loan'
then 'Title loan' 
end as title_loan_c,

-- personal loan
case when c.analyticalproduct = 'Personal loan'
then 'Personal loan' 
end as personal_loan_c,

-- payday loan
case when c.analyticalproduct = 'Payday loan'
then 'Payday loan' 
end as payday_loan_c,

-- federal student loan
case when c.analyticalsubproduct = 'Federal student loan'
then 'Federal Student loan' 
end as federal_student_loan_c,

-- private student loan
case when c.analyticalsubproduct = 'Private student loan'
then 'Private Student loan' 
end as private_student_loan_c,

-- credit card
case when c.analyticalproduct = 'Credit card' 
then 'Credit card' 
end as credit_card_c,

-- deposit
case when c.analyticalproduct = 'Checking or savings' 
then 'Deposit account' 
end as deposit_c,

-- credit reporting 3ncrc
case when c.analyticalproduct = 'Credit or consumer reporting'
and c.matched_company in ('REDACT')
then 'Credit reporting 3NCRC' 
end as credit_reporting_3ncrc_c,

-- credit reporting, small crc
case when c.analyticalproduct = 'Credit or consumer reporting'
and c.matched_company in ('REDACT')
then 'Credit reporting-small CRC' 
end as credit_reporting_small_crc_c,

-- debt collection
case when c.analyticalproduct = 'Debt collection' 
then 'Debt collection' 
end as debt_collection_c,

-- money service
case when c.analyticalproduct = 'Money transfer or service, virtual currency'
then 'Money services' 
end as money_service_c,

-- prepaid card
case when c.analyticalproduct = 'Prepaid card' 
then 'Prepaid card' 
end as prepaid_card_c,

-- furnishing
case when c.analyticalissue in (select * from furnishing_issues) 
and c.matched_company not in (select * from furnishing_exclusion)
then 'Furnishing' 
end as furnishing_c

from complaint c
    )
```
