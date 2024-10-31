---
marp: false
---
# Analysis of Flight Routes: Identifying Busiest and Most Profitable Round Trip Routes

## Table of Contents

1. [Introduction](#introduction)
2. [Data Overview](#data-overview)
3. [Methodology](#methodology)
    - [3.1. Environment Setup](#31-environment-setup)
    - [3.2. Data Loading](#32-data-loading)
    - [3.3. Data Cleaning and Preprocessing](#33-data-cleaning-and-preprocessing)
    - [3.4. Creating Route Identifiers](#34-creating-route-identifiers)
    - [3.5. Calculating Costs and Revenue](#35-calculating-costs-and-revenue)
    - [3.6. Aggregating Data for Busiest Routes](#36-aggregating-data-for-busiest-routes)
    - [3.7. Aggregating Data for Profitable Routes](#37-aggregating-data-for-profitable-routes)
    - [3.8. Merging Aggregated Data](#38-merging-aggregated-data)
    - [3.9. Sorting and Selecting Top Routes](#39-sorting-and-selecting-top-routes)
    - [3.10. Breakeven Analysis](#310-breakeven-analysis)
    - [3.11. Visualization](#311-visualization)
4. [Key Performance Indicators (KPIs)](#key-performance-indicators-kpis)
---

## Introduction

The aviation industry thrives on efficient route management to maximize profitability and meet passenger demand. This analysis aims to identify the busiest and most profitable round trip routes within the first quarter of 2019 (1Q2019), recommend optimal routes for business expansion, and determine the breakeven points for potential investments. By leveraging large datasets and employing robust data processing techniques, this study provides actionable insights to enhance operational efficiency and financial performance.

---

## Data Overview

### Datasets Utilized

1. **Airport Codes (`Airport_Codes.csv`):**
    - **Description:** Contains comprehensive information about airports, including IATA codes, types, and geographical coordinates.
    - **Key Columns:**
        - `IATA_CODE`: Unique identifier for each airport.
        - `TYPE`: Category of the airport (e.g., `medium_airport`, `large_airport`, `small_airport`, `heliport`, `closed`).

2. **Flights Data (`Flights.csv`):**
    - **Description:** Details individual flights, capturing origins, destinations, delays, distances, and cancellation statuses.
    - **Key Columns:**
        - `ORIGIN`: IATA code of the origin airport.
        - `DESTINATION`: IATA code of the destination airport.
        - `DEP_DELAY`: Departure delay in minutes.
        - `ARR_DELAY`: Arrival delay in minutes.
        - `DISTANCE`: Distance of the flight in miles.
        - `CANCELLED`: Binary indicator (1 if canceled, 0 otherwise).

3. **Tickets Data (`Tickets.csv`):**
    - **Description:** Records ticket sales, including passenger counts, fares, and cancellation statuses.
    - **Key Columns:**
        - `ORIGIN`: IATA code of the origin airport.
        - `DESTINATION`: IATA code of the destination airport.
        - `PASSENGERS`: Number of passengers booked.
        - `ITIN_FARE`: Fare per passenger.
        - `CANCELLED`: Binary indicator (1 if canceled, 0 otherwise).

### Data Size & Scope

- **Time Period:** First Quarter of 2019 (1Q2019).
- **Volume:** Millions of records across all datasets.
- **Geographical Coverage:** Global, spanning multiple continents and countries.

---

## Methodology

The analysis follows a structured approach to handle large datasets efficiently, ensuring accurate and meaningful insights. Below are the detailed steps undertaken during the analysis.

### 3.1. Environment Setup

**Libraries Imported:**

- `pandas`: For data manipulation and analysis.
- `numpy`: For numerical operations.
- `collections`: Specifically `defaultdict` and `Counter` for aggregating data.
- `matplotlib.pyplot` & `seaborn`: For data visualization.
- `gc`: Garbage collector for memory management during large data processing.

**Code:**
```python
import pandas as pd
import numpy as np
from collections import defaultdict, Counter
import matplotlib.pyplot as plt
import seaborn as sns
import gc 
```

## 3.2. Data Loading

### File Paths Defined

To begin the analysis, we first define the file paths for the datasets. This ensures that the script can locate and access the necessary data files within the Kaggle environment.

```python
AIRPORT_CODES_FILE = '/kaggle/input/airlines/Airport_Codes.csv'
FLIGHTS_FILE = '/kaggle/input/airlines/Flights.csv'
TICKETS_FILE = '/kaggle/input/airlines/Tickets.csv'
```

### Loading Airport Codes
We load the Airport_Codes.csv file using pandas. This dataset contains vital information about each airport, including their IATA codes and types.

```python
airport_codes = pd.read_csv(AIRPORT_CODES_FILE)
```

## 3.3. Data Cleaning and Preprocessing
### Filtering Airports with Valid IATA Codes
Since `ORIGIN` and `DESTINATION` in Flights and Tickets datasets reference IATA codes, it's essential to exclude airports without a valid `IATA_CODE`.

```python
airport_codes = airport_codes.dropna(subset=['IATA_CODE'])
```

### Creating Airport Cost Mapping
We create a mapping of airport operational costs based on the type of airport. This mapping will be used later to calculate the total cost associated with each flight route.

- Assumptions:
    - `medium_airport`: $5,000
    - `large_airport`: $10,000
    - Other types (`small_airport`, `heliport`, etc.): Default to $5,000

```python
airport_costs = {}
for _, row in airport_codes.iterrows():
    airport_code = row['IATA_CODE']
    size = row['TYPE'].lower()
    if size == 'medium_airport':
        airport_costs[airport_code] = 5000
    elif size == 'large_airport':
        airport_costs[airport_code] = 10000
    else:
        airport_costs[airport_code] = 5000
```

### Verifying Airport Costs Mapping
We verify a sample of the airport costs mapping to ensure correctness.

```python
print("\n Airport Costs Mapping Sample:")
sample_airports = list(airport_costs.items())[:5]
print(sample_airports)
```

## 3.4. Creating Route Identifiers
### Objective: 
Standardize route identifiers to ensure consistency across datasets.

### Approach:
Combine `ORIGIN` and `DESTINATION` IATA codes in alphabetical order.
This standardization treats routes like `JFK-ORD` and `ORD-JFK` as the same.

```python
def create_route(origin, destination):
    return '-'.join(sorted([origin, destination]))
```

## 3.5. Calculating Costs and Revenue
### Cost Calculation
The total cost for a flight encompasses variable costs (fuel, maintenance, crew), airport operational costs, and delay-related costs.

```python
def calculate_cost(row, airport_costs):
    distance = row['DISTANCE']
    origin = row['ORIGIN']
    destination = row['DESTINATION']
    
    # Variable Costs
    fuel_maintenance_crew = 8 * distance
    depreciation_insurance_other = 1.18 * distance
    total_variable_cost = fuel_maintenance_crew + depreciation_insurance_other
    
    # Airport Operational Costs
    origin_cost = airport_costs.get(origin, 5000)
    destination_cost = airport_costs.get(destination, 5000)
    total_airport_cost = origin_cost + destination_cost
    
    # Delays
    dep_delay = row['DEP_DELAY']
    arr_delay = row['ARR_DELAY']
    
    dep_delay_cost = ((dep_delay - 15) * 75) if dep_delay > 15 else 0
    arr_delay_cost = ((arr_delay - 15) * 75) if arr_delay > 15 else 0
    
    total_delay_cost = dep_delay_cost + arr_delay_cost
    
    # Total Cost
    total_cost = total_variable_cost + total_airport_cost + total_delay_cost
    return total_cost
```

### Revenue Calculation
Revenue is derived from passenger fares and baggage fees.

```python
def calculate_revenue(row):
    passengers = row['PASSENGERS']
    itin_fare = row['ITIN_FARE']
    
    # Revenue from fares
    revenue_fares = passengers * itin_fare
    
    # Baggage Fees
    baggage_fee = 70 * passengers * 0.5  # 50% passengers, $70 per round trip
    
    # Total Revenue
    total_revenue = revenue_fares + baggage_fee
    return total_revenue
```

## 3.6. Aggregating Data for Busiest Routes
### Objective: 
Identify the top 10 busiest round trip routes based on the number of flights.

### Approach:

1) **Initialize a Counter**: To keep track of flight counts per route. </br>
2) **Process Tickets Data in Chunks**: Due to large data size, process in manageable chunks.</br>
3) **Filter Canceled Flights**: Exclude flights where CANCELLED is 1. </br>
4) **Standardize Route Identifiers**: Using the create_route function. </br>
5) **Aggregate Flight Counts**: Update the Counter with the number of flights per route.

```python
busiest_routes_counter = Counter()
CHUNK_SIZE = 100000

for chunk in pd.read_csv(TICKETS_FILE, chunksize=CHUNK_SIZE):
    if 'CANCELLED' in chunk.columns:
        chunk = chunk[chunk['CANCELLED'] == 0]
    if {'ORIGIN', 'DESTINATION'}.issubset(chunk.columns):
        chunk['ROUTE'] = chunk.apply(lambda row: create_route(row['ORIGIN'], row['DESTINATION']), axis=1)
        route_counts = chunk['ROUTE'].value_counts()
        busiest_routes_counter.update(route_counts.to_dict())
    del chunk
    gc.collect()

top_10_busiest = busiest_routes_counter.most_common(10)

print("### Top 10 Busiest Round Trip Routes (1Q2019):")
for route, count in top_10_busiest:
    print(f"{route}: {count} round trip flights")
```

### 3.7. Aggregating Data for Profitable Routes

**Objective:**  
Determine the top 10 most profitable round trip routes by analyzing ticket sales and flight operational costs.

**Approach:**

1. **Initialize Dictionaries:**
   - **Purpose:** To store aggregated data such as the total number of passengers, total revenue, total costs, and the number of flights per route.
   - **Tools Used:** `defaultdict` from the `collections` module for efficient data aggregation.

2. **Process Tickets Data in Chunks:**
   - **Reason:** Handling large datasets by processing them in smaller, manageable chunks (`CHUNK_SIZE`) to optimize memory usage.
   - **Steps:**
     - **Filter Canceled Flights:** Exclude records where the `CANCELLED` column is `1` to ensure only active flights are considered.
     - **Convert Columns to Numeric:** Ensure that `ITIN_FARE` and `PASSENGERS` are numeric. Non-convertible values are set to `NaN` and subsequently dropped.
     - **Handle Missing Values:** Drop rows with `NaN` values in critical columns (`ITIN_FARE`, `PASSENGERS`) to maintain data integrity.
     - **Standardize Route Identifiers:** Use the `create_route` function to ensure consistency in route naming.
     - **Aggregate Passengers and Revenue:**
       - **Passengers:** Sum the number of passengers per route.
       - **Revenue:** Calculate revenue by multiplying the average fare (`ITIN_FARE`) by the number of passengers and adding baggage fees.
   
   - **Code:**
     ```python
     ticket_passengers = defaultdict(int)
     ticket_revenue = defaultdict(float)
     CHUNK_SIZE = 500000

     for chunk in pd.read_csv(TICKETS_FILE, chunksize=CHUNK_SIZE):
         if 'CANCELLED' in chunk.columns:
             chunk = chunk[chunk['CANCELLED'] == 0]
         
         required_columns = {'ORIGIN', 'DESTINATION', 'PASSENGERS', 'ITIN_FARE'}
         if required_columns.issubset(chunk.columns):
             chunk['ITIN_FARE'] = pd.to_numeric(chunk['ITIN_FARE'], errors='coerce')
             chunk['PASSENGERS'] = pd.to_numeric(chunk['PASSENGERS'], errors='coerce')
             
             initial_rows = chunk.shape[0]
             chunk = chunk.dropna(subset=['ITIN_FARE', 'PASSENGERS'])
             final_rows = chunk.shape[0]
             dropped_rows = initial_rows - final_rows
             if dropped_rows > 0:
                 print(f"Dropped {dropped_rows} rows due to invalid 'ITIN_FARE' or 'PASSENGERS'")
             
             chunk['ROUTE'] = chunk.apply(lambda row: create_route(row['ORIGIN'], row['DESTINATION']), axis=1)
             
             route_group = chunk.groupby('ROUTE').agg({
                 'PASSENGERS': 'sum',
                 'ITIN_FARE': 'mean'
             }).reset_index()
             
             for _, row_data in route_group.iterrows():
                 route = row_data['ROUTE']
                 passengers = row_data['PASSENGERS']
                 avg_fare = row_data['ITIN_FARE']
                 revenue = (avg_fare * passengers) + (70 * passengers * 0.5)  # Baggage fees
                 ticket_passengers[route] += passengers
                 ticket_revenue[route] += revenue
         
         del chunk
         gc.collect()

     tickets_aggregated = pd.DataFrame({
         'ROUTE': list(ticket_passengers.keys()),
         'TOTAL_PASSENGERS': list(ticket_passengers.values()),
         'TOTAL_REVENUE': list(ticket_revenue.values())
     })
     
     print("\n### Aggregated Ticket Data Sample:")
     display(tickets_aggregated.head())
     ```

3. **Aggregate Flights Data:**
   - **Purpose:** To calculate the total operational costs and the number of flights per route.
   - **Steps:**
     - **Filter Canceled Flights:** Exclude records where the `CANCELLED` column is `1`.
     - **Convert Columns to Numeric:** Ensure that `DEP_DELAY`, `ARR_DELAY`, and `DISTANCE` are numeric. Non-convertible values are set to `NaN` and subsequently dropped.
     - **Handle Missing Values:** Drop rows with `NaN` values in critical columns (`DEP_DELAY`, `ARR_DELAY`, `DISTANCE`) to maintain data integrity.
     - **Standardize Route Identifiers:** Use the `create_route` function to ensure consistency in route naming.
     - **Calculate Cost per Flight:** Use the `calculate_cost` function to determine the total cost associated with each flight.
     - **Aggregate Costs and Flight Counts:** Sum the total costs and count the number of flights per route.
   
   - **Code:**
     ```python
     flight_costs = defaultdict(float)
     flight_counts = defaultdict(int)
     
     for chunk in pd.read_csv(FLIGHTS_FILE, chunksize=CHUNK_SIZE):
         if 'CANCELLED' in chunk.columns:
             chunk = chunk[chunk['CANCELLED'] == 0]
         
         required_columns = {'ORIGIN', 'DESTINATION', 'DEP_DELAY', 'ARR_DELAY', 'DISTANCE'}
         if required_columns.issubset(chunk.columns):
             chunk['DEP_DELAY'] = pd.to_numeric(chunk['DEP_DELAY'], errors='coerce')
             chunk['ARR_DELAY'] = pd.to_numeric(chunk['ARR_DELAY'], errors='coerce')
             chunk['DISTANCE'] = pd.to_numeric(chunk['DISTANCE'], errors='coerce')
             
             initial_rows = chunk.shape[0]
             chunk = chunk.dropna(subset=['DEP_DELAY', 'ARR_DELAY', 'DISTANCE'])
             final_rows = chunk.shape[0]
             dropped_rows = initial_rows - final_rows
             if dropped_rows > 0:
                 print(f"Dropped {dropped_rows} rows due to invalid 'DEP_DELAY', 'ARR_DELAY', or 'DISTANCE'")
             
             chunk['ROUTE'] = chunk.apply(lambda row: create_route(row['ORIGIN'], row['DESTINATION']), axis=1)
             
             chunk['COST'] = chunk.apply(lambda row: calculate_cost(row, airport_costs), axis=1)
             
             route_group = chunk.groupby('ROUTE').agg({
                 'COST': 'sum',
                 'ROUTE': 'count'
             }).rename(columns={'ROUTE': 'FLIGHTS'}).reset_index()
             
             for _, row_data in route_group.iterrows():
                 route = row_data['ROUTE']
                 flight_costs[route] += row_data['COST']
                 flight_counts[route] += row_data['FLIGHTS']
         
         # Free up memory
         del chunk
         gc.collect()
     
     flights_aggregated = pd.DataFrame({
         'ROUTE': list(flight_costs.keys()),
         'TOTAL_COST': list(flight_costs.values()),
         'FLIGHTS': list(flight_counts.values())
     })
     
     print("\n### Aggregated Flights Data Sample:")
     display(flights_aggregated.head())
     ```

4. **Merge Aggregated Data:**
   - **Purpose:** To combine ticket sales data with flight operational costs, enabling the calculation of profitability per route.
   - **Steps:**
     - **Merge DataFrames:** Use an inner join on the `ROUTE` column to ensure only routes present in both datasets are included.
     - **Calculate Profit:** Subtract `TOTAL_COST` from `TOTAL_REVENUE` to determine the profit for each route.
   
   - **Code:**
     ```python
     merged_aggregated = pd.merge(tickets_aggregated, flights_aggregated, on='ROUTE', how='inner')
     
     # Calculate Profit
     merged_aggregated['PROFIT'] = merged_aggregated['TOTAL_REVENUE'] - merged_aggregated['TOTAL_COST']
     
     print("\n### Merged Aggregated Data Sample:")
     display(merged_aggregated.head())
     ```

5. **Sort and Select Top 10 Profitable Routes:**
   - **Purpose:** To identify the top 10 routes with the highest profitability.
   - **Steps:**
     - **Sort DataFrame:** Order the merged data in descending order based on the `PROFIT` column.
     - **Select Top 10:** Extract the first 10 entries from the sorted DataFrame.
   
   - **Code:**
     ```python
     top_10_profitable = merged_aggregated.sort_values(by='PROFIT', ascending=False).head(10)
     
     print("\n Top 10 Most Profitable Round Trip Routes (1Q2019):")
     display(top_10_profitable[['ROUTE', 'PROFIT', 'TOTAL_REVENUE', 'TOTAL_COST', 'FLIGHTS']])
     ```

6. **Output and Interpretation:**
   
   - **Interpretation:**
     - **High Profitability:** The routes listed have generated significant profits, making them prime candidates for strategic focus.
     - **Revenue vs. Cost:** A substantial difference between `TOTAL_REVENUE` and `TOTAL_COST` indicates effective cost management and strong revenue generation.
     - **Flight Volume:** The number of flights (`FLIGHTS`) provides context on the scale of operations required to achieve the reported profits.

## 3.10. Breakeven Analysis
### Objective: 
Determine the number of round trip flights required to cover the upfront airplane cost of $90 million for each recommended route.

### Approach:

1) **Define Upfront Cost**: Set a constant value representing the investment per route. </br>
2) **Calculate Breakeven Flights**: Divide the upfront cost by the profit per flight for each route. </br>

```python
recommended_routes = top_10_profitable.head(5).copy()

print("### Recommended 5 Round Trip Routes:")
display(recommended_routes[['ROUTE', 'PROFIT', 'TOTAL_REVENUE', 'TOTAL_COST', 'FLIGHTS']])

UPFRONT_COST = 90000000  # $90 million

recommended_routes['BREAKEVEN_FLIGHTS'] = recommended_routes['PROFIT'].apply(lambda x: UPFRONT_COST / x if x > 0 else np.nan)

print("### Breakeven Analysis for Recommended Routes:")
display(recommended_routes[['ROUTE', 'BREAKEVEN_FLIGHTS', 'PROFIT', 'TOTAL_REVENUE', 'TOTAL_COST', 'FLIGHTS']])
```

## 3.11. Visualization
### Objective: 
Visualize the top 10 most profitable routes to provide a clear and intuitive understanding of the data.

### Approach:

- **Bar Chart**: Utilize seaborn to create a bar chart representing the profitability of the top 10 routes. </br>
- **Customization**: Adjust figure size, labels, and title for clarity.

```python
plt.figure(figsize=(12, 6))
sns.barplot(x='ROUTE', y='PROFIT', data=top_10_profitable)
plt.title('Top 10 Most Profitable Round Trip Routes (1Q2019)')
plt.xlabel('Route')
plt.ylabel('Profit ($)')
plt.xticks(rotation=45)
plt.show()
```

# Key Performance Indicators (KPIs)

To ensure sustained success and monitor the effectiveness of the recommended routes, the following KPIs are essential:

## 1. On-Time Performance (OTP)

- **Definition:**  
  Percentage of flights that depart and arrive on time.

## 2. Load Factor

- **Definition:**  
  Percentage of occupied seats per flight.

- **Importance:**  
  Measures efficiency in utilizing aircraft capacity and affects profitability.

## 3. Revenue per Available Seat Mile (RASM)

- **Definition:**  
  Revenue generated per seat per mile.

- **Importance:**  
  Indicates profitability relative to distance and capacity.

## 4. Cost per Available Seat Mile (CASM)

- **Definition:**  
  Operating costs per seat per mile.

- **Importance:**  
  Helps in monitoring and controlling operational costs.

## 5. Flight Cancellation Rate

- **Definition:**  
  Percentage of flights canceled.

- **Importance:**  
  Affects reliability and customer trust.

## 6. Average Delay Minutes

- **Definition:**  
  Average minutes of delay per flight.

- **Importance:**  
  Impacts operational efficiency and customer satisfaction.

## 7. Profit Margin per Route

- **Definition:**  
  Percentage of profit relative to revenue for each route.

- **Importance:**  
  Identifies the most and least profitable routes.

---