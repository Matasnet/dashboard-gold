# Gold Prices Dashboard

This project is a Dash application that visualizes and analyzes gold prices using data from the National Bank of Poland (NBP).

## Features

- **Date Range Selection**: Users can select a range of dates to display the gold price data.
- **Gold Price Chart**: A tab to display a line chart of gold prices for the selected date range.
- **Gold Price Analysis**: A tab that provides a statistical analysis of gold prices including average, minimum, maximum, quartiles, mode, standard deviation, and interquartile range.
- **Footer**: The application includes a footer with contact information and the data source link.

## Requirements

- Python 3.x
- Dash
- Plotly
- Pandas
- Matplotlib
- Requests

## Installation

Install the required packages:

```sh
pip install dash plotly pandas matplotlib requests
```

## Running the Application

Run the following command to start the server:

```sh
python main.py
```

Then, open your web browser and go to `http://127.0.0.1:8050/` to view the application.

## Code Explanation

### Importing Libraries

The application imports the necessary libraries for creating the dashboard, fetching data, and performing statistical analysis:

```python
from datetime import datetime, date
from io import BytesIO
import base64

import dash
from dash import dcc, html, Input, Output
import requests
import pandas as pd
import matplotlib.pyplot as plt
```

### Fetching Data

The `get_gold_data` function fetches gold price data from the NBP API for a specified date range and processes it into a pandas DataFrame:

```python
def get_gold_data(start_date, end_date):
    url = f'https://api.nbp.pl/api/cenyzlota/{start_date}/{end_date}/'
    try:
        response = requests.get(url)
        response.raise_for_status()
        data = response.json()

        dates = []
        prices = []
        for item in data:
            dates.append(datetime.strptime(item['data'], '%Y-%m-%d').date())
            prices.append(float(item['cena']))

        df = pd.DataFrame({'Date': dates, 'Price': prices})
        df['Date'] = pd.to_datetime(df['Date'])
        return df
    except requests.exceptions.RequestException as e:
        print(f'Error fetching data: {e}')
        return None
```

### Application Layout

The layout of the Dash application includes a header, date picker, tabs for different views, and a footer:

```python
app.layout = html.Div([
    html.H2('Gold Prices | Data from NBP!', style={'textAlign': 'center'}),

    html.Div([
        html.Label('Select date range:'),
        dcc.DatePickerRange(
            id='date-picker-range',
            start_date='2024-01-01',
            end_date=date.today().strftime("%Y-%m-%d"),
            display_format='YYYY-MM-DD'
        )
    ], style={'textAlign': 'center', 'margin-bottom': '20px'}),

    dcc.Tabs(
        id='tabs-1',
        children=[
            dcc.Tab(label='Gold Chart for Selected Period', value='gold'),
            dcc.Tab(label='Gold Price Analysis', value='analysis'),
        ],
        value='gold'
    ),

    html.Div(id='div-1', style={'padding-bottom': '100px'}),

    html.Div([
        html.Hr(),
        html.P('This is a footer text.'),
        html.P('Contact: contact@example.com'),
        html.P([
            "Data source: ",
            html.A("NBP", href="https://nbp.pl/en/", target="_blank")
        ]),
    ], style={'textAlign': 'center', 'padding': '10px', 'position': 'fixed', 'left': '0', 'bottom': '0', 'width': '100%', 'backgroundColor': '#f1f1f1', 'zIndex': '1000'})
])
```

### Callbacks

The callback function updates the content of the page based on the selected tab and date range:

```python
@app.callback(
    [Output('div-1', 'children')],
    [Input('tabs-1', 'value'),
     Input('date-picker-range', 'start_date'),
     Input('date-picker-range', 'end_date')]
)
def render_content(tab, start_date, end_date):
    df = get_gold_data(start_date, end_date)

    if df is None:
        return [html.Div([html.P('Error fetching data. Please try again later.')])]

    if tab == 'gold':
        fig, ax = plt.subplots(figsize=(13, 6))
        ax.plot(df['Date'], df['Price'], label='Actual Price')
        ax.set_xlabel('Date')
        ax.set_ylabel('Gold Price')
        ax.set_title('Gold Prices from NBP')
        ax.legend()
        ax.grid(True)

        buf = BytesIO()
        fig.savefig(buf, format='png')
        buf.seek(0)
        image_base64 = base64.b64encode(buf.read()).decode('utf-8')
        buf.close()

        plt.close(fig)

        avg_price = round(df['Price'].mean(), 2)
        max_price = round(df['Price'].max(), 2)
        min_price = round(df['Price'].min(), 2)

        return [
            html.Div([
                html.Img(src='data:image/png;base64,{}'.format(image_base64), style={'width': '100%'}),
                html.Div([
                    html.P(f"Data period: from {start_date} to {end_date}"),
                    html.P(f"Average Price: {avg_price} PLN"),
                    html.P(f"Maximum Price: {max_price} PLN"),
                    html.P(f"Minimum Price: {min_price} PLN")
                ], style={'textAlign': 'center', 'margin-top': '20px'})
            ])
        ]
    elif tab == 'analysis':
        avg_price = round(df['Price'].mean(), 2)
        min_price = round(df['Price'].min(), 2)
        max_price = round(df['Price'].max(), 2)
        q1 = round(df['Price'].quantile(0.25), 2)
        q2 = round(df['Price'].quantile(0.5), 2)
        q3 = round(df['Price'].quantile(0.75), 2)
        mode = round(df['Price'].mode()[0], 2)
        std_dev = round(df['Price'].std(), 2)
        iqr = round(q3 - q1, 2)

        stats_df = pd.DataFrame({
            'Statistic': ['Average Price', 'Minimum Price', 'Maximum Price', 'First Quartile', 'Median', 'Third Quartile', 'Mode', 'Standard Deviation', 'Interquartile Range'],
            'Value': [avg_price, min_price, max_price, q1, q2, q3, mode, std_dev, iqr],
            'Definition': [definitions['Average Price'], definitions['Minimum Price'], definitions['Maximum Price'], definitions['First Quartile'], definitions['Median'], definitions['Third Quartile'], definitions['Mode'], definitions['Standard Deviation'], definitions['Interquartile Range']]
        })

        html_table = html.Table([
            html.Thead(
                html.Tr([html.Th(col) for col in stats_df.columns])
            ),
            html.Tbody([
                html.Tr([
                    html.Td(stats_df.iloc[i][col]) for col in stats_df.columns
                ]) for i in range(len(stats_df))
            ])
        ], className='table table-striped')

        return [
            html.Div([
                html.H3('Gold Price Analysis'),
                html.Div([
                    html.P(f"Data period: from {start_date} to {end_date}"),
                    html_table
                ], className='analysis-table')
            ])
        ]

if __name__ == '__main__':
    app.run_server(debug=True)
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
```
