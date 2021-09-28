## Project 1 - Additive regression complaint forecast model

#### *Languages and Tools:*
* Python
  * os
  * datetime
  * pandas
  * fbprophet
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






* Plotly
```python3
# Interactive Bar Chart
fig5 = make_subplots(rows=2, cols=1)

fig5.append_trace(go.Bar(
    x=list(v_amp_yes_gene_in_ct['GeneName']),
    y=list(v_amp_yes_gene_in_ct['Count']),
    name='"All" (Amp)',
    marker_color='rgb(95, 182, 239)',
    width=0.4
), row=1, col=1)

fig5.append_trace(go.Bar(
    x=list(v_amp_no_gene_in_ct['GeneName']),
    y=list(v_amp_no_gene_in_ct['Count']),
    name='"All" (No Amp)',
    marker_color='rgb(204, 204, 255)',
    width=0.4
), row=1, col=1)

fig5.append_trace(go.Bar(
    x=list(v_amp_yes_gene_ct['GeneName']),
    y=list(v_amp_yes_gene_ct['Count']),
    name='"In" (Amp)',
    marker_color='rgb(168, 241, 115)',
    width=0.4
), row=2, col=1)

fig5.append_trace(go.Bar(
    x=list(v_amp_no_gene_ct['GeneName']),
    y=list(v_amp_no_gene_ct['Count']),
    name='"In" (No Amp)',
    marker_color='rgb(255, 176, 140)',
), row=2, col=1)

fig5.update_layout(
    # template='plotly_dark',
    legend_title_text='Status',
    title="Fig. 5: Gene Counts",
    yaxis_title="Count",
    font=dict(
        size=14,
    ))

# change the bar mode
fig5.update_layout(barmode='group')

pio.write_image(fig5, 'fig5.jpeg', scale=20)
fig5.show()
```

### Random Forest
* scikit-learn
* Matplotlib
```python3
# Selecting important features, creating a Random Forest model, and assessing with a ROC curve
df = df
target = 'target'

numeric_features = df.select_dtypes(include=['int64', 'float64']).drop(target, axis=1).columns
categorical_features = df.select_dtypes(include=['object']).columns

X = df.drop([target, 'GC', 'AT', 'nt_length', 'homopol_count'], axis=1)
y = df[target]

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, stratify=df[target], random_state = 100, shuffle=True)

# exhaustive grid search
model_RF = RandomForestClassifier()
n_estimators = [10, 50, 100]
max_features = ['sqrt', 'log2']
# define grid search
grid = dict(n_estimators=n_estimators,max_features=max_features)
cv = RepeatedStratifiedKFold(n_splits=10, n_repeats=3, random_state=1)
grid_search = GridSearchCV(estimator=model_RF, param_grid=grid, cv=cv, scoring='roc_auc')
grid_result_RF = grid_search.fit(X, y)
# summarize results
print("Best from ALL Features: %f using %s" % (grid_result_RF.best_score_, grid_result_RF.best_params_))
means = grid_result_RF.cv_results_['mean_test_score']
stds = grid_result_RF.cv_results_['std_test_score']
params = grid_result_RF.cv_results_['params']
for mean, stdev, param in zip(means, stds, params):
    print("%f (%f) with: %r" % (mean, stdev, param))

# best model from grid search
RF = RandomForestClassifier(**grid_result_RF.best_params_)
RF.fit(X_train, y_train)

y_pred_RF=RF.predict(X_test)
y_pred_RF_train=RF.predict(X_train)

# get top 20 features
feat_importances = pd.Series(RF.feature_importances_, index=X.columns)
feat_importances.nlargest(19).plot(kind='barh')
plt.savefig('imp_feats.png')

# subset to top 15 features
feature_list = list(X)
indices = np.argsort(feat_importances)[::-1]
ranked_index_2 = [feature_list[i] for i in indices]

imp_feats = ranked_index_2[0:16]

# AUROC
rfc_pred = rfc.predict_proba(X_test)
false_positive_rateRF, true_positive_rateRF, thresholdRF = roc_curve(y_test, rfc_pred[:,1])
roc_aucRF = auc(false_positive_rateRF, true_positive_rateRF)
plt.figure(figsize = (8,8))
plt.title('Receiver Operating Characteristic')
plt.plot(false_positive_rateRF, true_positive_rateRF, color = 'green', label = 'RF AUC = %0.2f' % roc_aucRF)
plt.legend(loc = 'lower right')
plt.plot([0, 1], [0, 1], linestyle = '--')
plt.axis('tight')
plt.ylabel('True Positive Rate')
plt.xlabel('False Positive Rate')
plt.savefig('rf_roc_auc.png')
```
### Write Report
* HTML
* Python
```html
HTML script for writing final report

html_string = '''
<html>
    <head>
      <script src="https://cdn.plot.ly/plotly-latest.min.js"></script>
      <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.1/css/bootstrap.min.css">
      <style>body{ margin:0 100; background:Gainsboro; }</style>

      <style>
      img {
      display: block;
      margin-left: auto;
      margin-right: auto;}
      </style>

      </head>
      <body>
      <p style="text-align:center;font-size:38px;">Redacted</p>

      <p style="text-align:center;font-size:25px;">Data</p>

      <p style="text-align:center;font-size:20px;">Raw Data</p>
      <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
      <div style="overflow-x:auto;">
      ''' + summary_table_1 + '''
      </div>

      <p style="text-align:center;font-size:20px;">Transformed Data</p>
      <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
      <div style="overflow-x:auto;">
      ''' + summary_table_2 + '''
      </div>


      <p style="text-align:center;font-size:25px;">Exploratory Analysis</p>

      <p style="text-align:center;font-size:25px;">Figure 1</p>
      <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
      ''' + fig1_html + '''
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>

      <p style="text-align:center;font-size:25px;">Figure 2</p>
      <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
      ''' + fig2_html + '''
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>

      <p style="text-align:center;font-size:25px;">Figure 3</p>
      <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
      ''' + fig3_html + '''
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>

      <p style="text-align:center;font-size:25px;">Figure 4</p>
      <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
      ''' + fig4_html + '''
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>

      <p style="text-align:center;font-size:25px;">Figure 5</p>
      <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
      ''' + fig5_html + '''
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>

      <p style="text-align:center;font-size:25px;">Figure 6</p>
      <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
      ''' + fig6_html + '''
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>

      <p style="text-align:center;font-size:25px;">Figure 7</p>
      <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
      ''' + fig7_html + '''
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>

    <p style="text-align:center;font-size:25px;">Build a Random Forest Classifier</p>

        <p style="text-align:center;font-size:25px;">Correlation Plot of Data</p>
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
    <img src="../PROJECTS/corr_plot.png" style="width:50%">
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>


    <p style="text-align:center;font-size:25px;">Top Twenty Features from Classifier</p>
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
    <img src="../PROJECTS/rf_imp_feats.png" style="width:50%">
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>


    <p style="text-align:center;font-size:25px;">AUROC of Random Forest Classifier</p>
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>
    <img src="../PROJECTS/rf_roc_auc.png" style="width:50%">
    <p style="text-align:center;font-size:16px;">Stuff about the figure.</p>


    </body>
</html>'''

```
```python3
# write selected results to HTML filename
with open("proj_report.html", 'w') as f:
    f.write(html_string)
```
