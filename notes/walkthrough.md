## Working notes written while building out the model

## Section 1: Set up and data download

Yf.download is how I am getting my daily data
I have the tickers, and the start and end dates I want to get all the data from

Having raw = yf.download(...) is going to give me the open, high , low, close, volume etc
- I just want the close price for each day so I am going to do prices = raw[“Close”]
- The rows are the trading days and the columns are my tickers (for ^TNX its thats days yield
- Should be about 5 years (1250 rows)
- prices.tail() gives me the last 5 (.head() would give me the first 5)
- For the 10 year treasury yield, 4.402 means 4.402%

## Section 2: Transforming

We’re converting prices into units for the regression to actually use

Tickers = [....] defines the whole portfolio once so I can go back to it for loops and weights instead of retyping it.

Because prices has the 5 tickers in them, we can just do prices[tickers]
- Because we are comparing it to SPY as our proxy but dont want it in the tickers list because it isnt one of our 5 equities, we just add it like [“SPY”] because that is how it is in prices
    - prices[tickers + [“SPY”]]
- We want the percentage change of each so we add .pct_change() to the end to receive the percentage
    - Because it gives us a decimal, we want to multiply it by 100 like “8 * 100”
- We want returns over prices because day to day changes carry real information unlike pure prices (also allows us to compare better)

We also need to find the daily yield (we will use basis points instead of %)
- We use .diff() instead of .pct_change() because the yield is already a % and doing a % of a % doesnt work here.
    - Example: if Day 1 is 4.4 for the yield and Day 2 is 4.45, the difference is 0.05. This is the direct change in yield we need instead of a % change of the yield
- Again, we multiply by 100 to get the yield in basis points and not just a fraction

From there, we need a data frame (df) to put this all together
- We use pandas for this because it gives us that structure for the spreadsheet with all our data
- To connect the daily basis points to the rest of the data, we need to use concat to concatenate both the returns and d_bps.
    - Because d_bps would show up as ^TNX in the column header, we need to use .rename() to give it a new name. In our example, we changed it to “d_bps” of course.
    - The axis is the direction you grow the table
        - axis=1 gives makes sure that the yield column goes next to the return columns (good for different measurements, same dates)
        - axis=0 stacks them on top of eachother. It would put our yield table underneath the returns (kind of like adding more rows). It would have twice as many rows and every row would have mostly holes.(good for same columns, longer history)
    - We need to drop the first day of the yield change because the first day has no day before for a change. .dropna() allows that to happen.

The dataframe (df) will give us our regression ready dataset
- Each row is a trading day and each column is the 7 inputs (5 equities plus the comparisons (market and yield)).
    - When we print the shape, we get (1253,7) which pairs the trading days and inputs
- We want a way to get all our statistical measures ready for our regression, so we use the .describe() feature to get all our numbers (count, mean, std, min, percentiles, and max)

While this is the data for the daily returns, we may want the data for the weekly returns
- Most of the code would be the same but there are a few changes
    - 1. While we still use our same tickers list, we need to convert the prices from daily to weekly.
        - That is where we use the resample function. This function tells pandas to regroup our daily data into weekly data for fridays close. The W stands for weekly and FRI is for friday. You can pick any weekday. Monthly and daily doesnt need a resample (in our case)
        - .last() tells us what to do with our new buckets and in this case, it is the closing snapshot. .first() would give us each weeks opening snapshot instead (which isnt helpful here).
    - 2. We change the names of each asssignment to the weekly title (_w to each)
- The shape will now give us (261,7) for each week (5 trading days)

## Section 3: The regression

For running the regression, we need a library for statistical modelling. This is where statsmodels is useful. We can run regressions, time series hypothesis testing, etc)
- .api is a submodule that gathers the most commly used pieces we would need

From our dataframe (df), we wnat to slice out the 2 columns we are comparing (market and yield)
- By having a list of the 2 explainers in the dataframe, we just return those 2 columns.

For our regression to work, we need a way to have a column for the intercept. Right now, we just have the SPY and d_bps column that will be connected to their beta coefficient but we dont have anything to multiply to our coefficient constant.
- To fix that, we use .add_constant() which allows for all the data in that column to be 1 all the way down
    - This is useful because it allows the coefficient constant to grab onto something and be just its coefficient constant for our intercept
        - Without this we would have our regression without an intercept (ie it would assume that the stock returns exactly 0% when other factors are flat)
        - It would also distort the beta as they would get bent to absorb any of the baseline drift
We need a way to then construct the model, so we use .OLS()
- In the parentheses, we place 2 arguments
    - 1. Y which is our thing we want to explain (in this case, the return of John Deere’s stock)
    - 2. X which is our explainers we created from the assignment above
- Our goal here is to explain DE using the other columns we have (the explainers)
Now that we set it up, we need to actually compute it
- .fit() executes the least squares math and allows to get an actual answer
    - Without it, the statement before is just a description of what to have but wouldn’t actually run
    - With it, it finds all 3 of our coefficients need for our regression (constant, SPY beta, and d_bps beta)
        - It finds the gaps between predicted and actual to get the Ordinary Least Squares (OLS)
Finally, we want a summary of what we computed to show up visually.
- .summary() allows us to see the whole OLS Regression Results

FYI- The reason there are 2 print statements for the yield and basis points is to double check the units and scaling to make sure the regression was accurate

If we wanted the regression in weekly formatting, all we have to do is use weekly dataframe (df_w) from before and change the assignments of X and model to XW and model_w for the weekly versions.

## Section 4: loops and table for betas

Before our loop, we need a way to hold all the key:value pairs from the loop.
- That is why we assign an empy dictionary to hold each thing we loop

Looping through each ticker in our tickers list
- We use 2 assignments from above to get each regression
    - The only difference is way generalize the ticker in the dataframe to work with each loop (ie df[ticker] instead of df[“DE”])
- The new assignment in the loop is getting the aspects from the results we need
    - The key value pairing allows us to give names to each of the data answers we need
        - We first need the market beta so we use m.params[“SPY”] to pull out the market beta coefficient from our regression 
        - We then do the exact same for our rate beta using m.params[“d_bps”] * 100
        - To get our standard errors for eachm we woudl use m.bse[] which stands for beta standard errors.
            - Because daily alpha is statistically almost 0 for everything and every large liquid company (all 5 here are) hav an overwelmingly high significance (of course), we aren’t going to run the standard erros for these
            - Instead, we will just find the standard error for the rate beta because that variable can be very different and not always significant
        - Same reason we are finding only the standard error for rate beta is the same reason we are only finding the t stat for the rate beta too.
        - We also want the R^2 so we use m.rsquared to receive that from our data

After we got all the data from our loopm we want to put it into a small dataframe to see the results
- We create a dataframe using pandas and use our results dictionary to show the values
    - The raw version will have tickers as columns so we use .T to transpose it to have the 5 tickers as rows (1 stock per row)
    - We then visually round it to 3 decimal places to trim up what we see

Then we just display the dataframe by outputting our assignment name (betas)

And again, we can do the same for weekly results by making sure the datafram is the weekly one (df_w) and making our dictionary assignment results_w instead of results

## Section 5: Portfolio aggregation 

We initially want to give each stock equal weighting to make it even. Because we have 5 stocks, each gets a 20% weight for our portfolio
- Using pandas, we need a constructor to build a series so we use pd.Series()
    - It allows us to make Series like prices[] and m.params from scratch
    - Because all our weights are equal, we just need 0.2 there
        - If they werent, we would need a list of each weights in the order of the tickers list
    - To know where its coming from, we show which index it is
        - In our case, it is the tickers list
    - Using a series also allows us to assign the values to names and not just positions

For our new regression of our full portfolio, we need a way to give each stock its weight and add them up to create the portfolio
- We would need a few new elements
    - 1. We need a new column for the full portfolio so we use df[“PORT”] as our portfolio dataframe
    - 2. Using of tickers dataframe, we use mul(weights) to multiply each of the weights to their respective stock to give us their weight in our portfolio
    - 3. We need to add them across each stock row, so we use .sum(axis=1) to add across. Itll add up each stock with all eachother for that specific day and go all the way down for all the days in our data, We arent adding new days to our data, so that is why it is axis=1 instead of axis=0.

- From there, we use the X assignment again to help build out our regression
- Instead of using our previous assignment to fit our regression, we use a modified form
    - Because we have the new portfolio column, we use that instead of going through a specfic stock

- Finally, we print out a summary of our portfolio regression


There are 2 routes to get the portfolios beta
- 1. Regress the blend directly (like we just did)
- 2. Blend each regression (add up the betas from our loops)

To show how each of the market and rate betas contribute to the portfolio, we create a contribution table to show it
- From our betas dataframe, we want the market anda rate beta from the dictionary
    - We use the .mul() again to multiply each weight (0.2 in our case) from its matching stock
        - Because the tickers are row labels now, we use axis=0 instead
- contribution.sum() allows us to sum of all the tables columns (ie add up all the market betas and rate betas)
    - To get the row to show up in our dataframe, we need the sum to be assigned by contribution.loc[“PORTFOLIO”] which adds in a new row called PORTFOLIO
        - It is the similar to how we added the “PORT” column to our dataframe (df[“PORT”]) but for rows instead of columns
        - .loc can do both (example: .iloc(row_label, column_label))

We finally round our dataframe 3 decimals for visual appeal


## Section 6: The Matrix

We got to the scenario analysis (“what if?”) section we have been building up to

In our 5x5 matrix, we have the market moves and yield moves in a list based off even swings up or down
- Spx_moves is quoted in percentage moves
- Bps_moves is in basis points moves (not percents)

We want to pull the portfolio parameters for the market beta and rate beta from m_port
- We will use SPY and daily bps from our portfolio regression (just like we pulled them for the loop)
- Because we have our bps_moves list already in bps, we dont need to multiply it by anything

To get our matrix into visual form (at first), we will build a dataframe for the matrix. 
- The double loop we have inside the dataframe is a list comprehension
    - This loop inside a loop allows us to get each of the 25 values we need
        - The inner piece builds a singular row
        - The outer piece builds out each column then on after
    - So we would get the row for -100 bps (and get all 5 market % values) and then move on over
- The index and columns parts of the dataframe give us the row and colum headers
    - F string allows us to insert in values and assignments into strings
        - The +d means an integer where you always show the sign (+)
        -Then you loop that for having the spx_moves in the row headers and bps_moves in the column headers

We then round our matrix and present it


Lastly, we want to visual our matrix a bit better than what we have with just the dataframe
- That is where matplotlib and seaborn come in as libraries
    - Matplotlib is pythons base plotting library
    - Seaborn is a layer on top that gives us nice chart types
- Seaborn gives us the heatmap and matplotlib gives us the labels around it

The variables inside the heatmap include the actual matrix we build with our dataframe and then a bunch of elements to make it look nice
- To print the number inside the colored cell, we use annot=True (without it, wed just have colors)
- To round in each cell, we use fmt=”.1f” for a float with 1 decimal place
- To give it the color we want, we use cmap to give us the colormap we want
    - In our case “RdYlGn” is red-yellow-green
- To make sure the values are centered, we use center=0 to keep it right in the middle
- To give the color bar a legend strip, we can use cbar_kws to access a small dictionary of options to show what the colorbar represents

Then we just plot the title, ylabel, and x label in the graph use plt.___(title)

Finally, we can present a finished polished scenario matrix of our portfolio using equal weightings against the market and 10 year yield.

