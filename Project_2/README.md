## Project 1 - Additive regression complaint forecast model

#### *Languages and Tools:*
* Python
  * os
  * datetime
  * pandas
  * fbprophet (Stan)
  * Plotly
  * psycopg2
* Bash
----------
### Makefile:
* Bash
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

### Prepare Dataframe:
* pandas
* psycopg2
```python3
pg_sslmode = environ.get("PGSSLMODE", "require")
pg_user = environ.get("PGUSER", None)
pg_password =  environ.get("PGPASSWORD", None)
pg_host = environ.get("PGHOST", None)
pg_port = environ.get("PGPORT", 5432)
pg_database = environ.get("PG_DATABASE", None)

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
    record = cursor.fetchone()
    return connection

# establish connection
connection = create_db_connection(pg_user, pg_password, pg_host, pg_port, pg_database, pg_sslmode)

# get table from database
postgreSQL_select_Query = "SELECT DATE(r.createddate) as day_of_createddate, \
                                CASE EXTRACT(DOW FROM r.createddate) \
                                                WHEN 0 THEN 'Sunday' \
                                                WHEN 1 THEN 'Monday' \
                                                WHEN 2 THEN 'Tuesday' \
                                                WHEN 3 THEN 'Wednesday' \
                                                WHEN 4 THEN 'Thursday' \
                                                WHEN 5 THEN 'Friday' \
                                                WHEN 6 THEN 'Saturday' \
                                END as weekday_of_createddate, \
                                DATE_TRUNC('week', r.createddate::date + 1):: date - 1 as week_of_createddate, \
                                COUNT(Distinct r.casenumber) as complaint_count \
                        FROM crdw.reporting r \
                        WHERE r.type = 'Mosaic Complaint' \
                        AND   (r.investigationdisposition <> 'Duplicate' or r.investigationdisposition is null) \
                        AND   (r.reason_close_with_no_action not in ('Duplicate (CFPB Spotted)', 'Duplicate (Company Spotted)') or r.reason_close_with_no_action is null) \
                        GROUP BY DATE(r.createddate), EXTRACT(DOW FROM r.createddate), DATE_TRUNC('week', r.createddate::date + 1):: date-1"


# get df from database
df1 = pd.read_sql_query(postgreSQL_select_Query, connection)

# select just the date and complaint columns
df1 = df1[['day_of_createddate', 'complaint_count']]

# drop the last row because it's the current date and contains incomplete reporting data
df1 = df1[:-1]

# rename the column headers to match the prophet naming convention
df1 = df1.rename(columns={'day_of_createddate': 'ds', 'complaint_count': 'y'})

# get date column in proper datetime format
df1['ds'] =  pd.to_datetime(df1['ds'])

# get past two years
two_yrs = (datetime.now() - timedelta(730)).strftime('%Y-%m-%d')

# filter df for past two years
df = df1[df1['ds'] >= two_yrs]

# get summary count of all complaints from 7/21/11 to two years ago
df_filt = df1[df1['ds'] < two_yrs]
df_filt_sum = sum(df_filt['y'])

# separate weekdays and weekends. these will be modeled separately then recombined.
weekdays = df[df['ds'].dt.dayofweek < 5]
weekends = df[df['ds'].dt.dayofweek > 5]

# get yesterday's date
yesterday = (datetime.now() - timedelta(1)).strftime('%Y-%m-%d')

# total complaint counts
print(f"Total complaints: {sum(df1['y'])}")

# calculate changepoints. these identify times when the probability distribution changes.
# noise detection for input as changepoints date
mean = df['y'].mean()
stdev = df['y'].std()

q1 = df['y'].quantile(0.25)
q3 = df['y'].quantile(0.75)
iqr = q3 - q1
high = mean + stdev
low = mean - stdev

# define this as changepoints in case you want to filter noise date using mean and standard deviation
df_filtered = df[(df['y'] > high) | (df['y'] < low)]
df_filtered_changepoints = df_filtered

#define this as changepoints in case you want to filter noise date using IQR
filtered_iqr = df[(df['y'] < q1 - (1.5 * iqr)) | (df['y'] < q3 + (1.5 * iqr)) ]
```

### Create Model:
* pandas
* fbprophet (Stan)
```python3
# params
# instantiate Prophet Object - weekdays
weekday_prophet = Prophet(
                  interval_width = 0.95,
                  daily_seasonality = False,
                  weekly_seasonality = True,
                  yearly_seasonality = True,
                  seasonality_mode='multiplicative',
                  changepoint_prior_scale = 0.5,
                  seasonality_prior_scale = 0.01,
                  holidays_prior_scale=10)

weekday_prophet.add_country_holidays(country_name='US')
#fit the model to training data , we try to use a whole of data as training data
weekday_prophet.fit(weekdays)

weekday_future = weekday_prophet.make_future_dataframe(periods = 120, freq = 'd')
weekday_forecast = weekday_prophet.predict(weekday_future)
weekday_forecast = weekday_forecast[weekday_forecast['ds'].dt.dayofweek < 5]


#instantiate Prophet Object - weekends
weekend_prophet = Prophet(
                  interval_width = 0.95,
                  daily_seasonality = False,
                  weekly_seasonality = True,
                  yearly_seasonality = True,
                  seasonality_mode='additive',
                  changepoint_prior_scale = 0.5,
                  seasonality_prior_scale = 0.01,
                  holidays_prior_scale=10)
    
weekend_prophet.add_country_holidays(country_name='US')
#fit the model to training data , we try to use a whole of data as training data
weekend_prophet.fit(weekends)

weekend_future = weekend_prophet.make_future_dataframe(periods = 120, freq = 'd')
weekend_forecast = weekend_prophet.predict(weekend_future)
weekend_forecast = weekend_forecast[weekend_forecast['ds'].dt.dayofweek > 5]

# round forecasted complaints to the nearest whole number
forecast = pd.concat([weekday_forecast, weekend_forecast], 0)
forecast = forecast.sort_values(by='ds', ascending=True)

forecast['yhat'] = forecast['yhat'].round(0).astype(int)
forecast['yhat_upper'] = forecast['yhat_upper'].round(0).astype(int)
forecast['yhat_lower'] = forecast['yhat_lower'].round(0).astype(int)
```

### Plot Forecast:
* Plotly
```python3
#plot the predicted value and observed value
fig1 = go.Figure([
    go.Scatter(x=df['ds'], y=df['y'], name='actual complaint data'),
    go.Scatter(x=forecast['ds'], y=forecast['yhat'], name='forecasted complaint data')
])

fig1.update_layout(legend=dict(
    yanchor="top",
    y=0.99,
    xanchor="left",
    x=0.005
))


fig1.update_layout(
    title= f"Daily complaint forecast (trained on data from {datetime.strptime(two_yrs, '%Y-%m-%d').strftime('%B %d, %Y')} to {datetime.strptime(yesterday, '%Y-%m-%d').strftime('%B %d, %Y')})",
    xaxis_title="Date (in days)",
    yaxis_title="Complaint counts (per day)"
)

fig1.show()
offline.plot(fig1, filename = INDEX_PATH, auto_open=False)

connection.close()
```
