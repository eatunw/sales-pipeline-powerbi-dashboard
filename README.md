# Sales pipeline dashboard in Power BI: spreading deal value across the months it gets delivered

Built by Emmanuel Atunde. Tools: Power BI, Power Query, DAX, Excel.

## The problem

Most pipeline reports I see make the same mistake. A deal worth $4,000 that runs from April to July gets dropped into April, so April looks huge and May to July look empty. Anyone using that report to plan staffing or cash flow is working from the wrong picture.

I wanted to build a pipeline and forecast dashboard that fixes this properly. Every deal's value should land in the months when the work actually happens, and the report should separate what is confirmed from what is still a maybe. It should also be honest about where the underlying data can't be trusted.

## The data

I used a public sales pipeline dataset from Kaggle. The raw file had 3,763 rows, but only 1,000 were real deals. The rest was blank padding. Each deal has an owner, customer, country, sales stage, deal size, a probability, a sales channel, a reporting date, an expected close date and a logged next step.

Two columns had no label. One turned out to be the number of days between the reporting date and the close date, which is redundant, so I dropped it. The other had values in only 10 rows with no pattern, so I dropped that too. There was also a stray footer row with nothing in it except a "1". I removed that as well.

## What I added, and what I didn't touch

The dataset only has one close date per deal. It has no project start and end dates, and without those you can't spread value across months. So I added them, and I want to be clear that this part is synthetic.

Project start is the close date plus a random lag of 0 to 14 days. Duration scales with deal size, with a bit of random jitter, and is capped between 1 and 8 months. End date is start plus duration. I fixed the random seed so the result can be reproduced. Everything else about the deals (owner, customer, stage, size, channel, country, next step) is original and unmodified.

I'm saying this up front because a real client's tender register would already contain real dates. The synthetic ones let me show the technique on realistic data. They don't tell you anything about real project timing.

## Cleaning in Power Query

Most of the cleaning was small. I set data types, converted the date columns from numbers to proper dates, and replaced the 173 empty Next Steps values with "Not Logged". Blank cells behave inconsistently in visuals and sometimes drop out of counts, so I wanted that group to show up as a category of its own.

The Power Query dialog won't match true null values if you leave the field empty. Typing the word null in the "Value to find" box is what works.

I also built a date table covering 1 January 2015 to 31 December 2016 (731 rows) and marked it as the model's date table.

## The part that took the most thought: spreading value across months

The fix is a bridge table with one row per deal per month the project is active. A deal running April to July becomes four rows, each carrying a quarter of the deal value. I built it in Power Query by working out how many months each project spans and generating a list of month start dates for each deal, then expanding that list into rows.

My first attempt returned errors on every row. The formula nested one `each` inside another, so the inner one grabbed the wrong reference. Pulling the start date and month count into named variables first fixed it:

```
let
    StartD = Date.StartOfMonth([Project Start Date]),
    MonthsCount = [Months Between]
in
    List.Transform({0..MonthsCount-1}, each Date.AddMonths(StartD, _))
```

The bridge table links to the main deal table by customer (every customer appears once in the source, so it works as a key) and to the date table by month. That second link is deliberately inactive, because the date table already connects to the deals through the reporting date. I turn the monthly link on only inside the measure that needs it, using USERELATIONSHIP. That lets one date table answer two different questions: when a deal was reported, and in which months its value lands.

I tested it by selecting 2016 in a year slicer. The deal table went blank, which is right because no deal was reported in 2016. The spread measure, though, showed 11,558 of value falling in early 2016, from projects that start in late 2015 and finish after the new year. Across both years the spread adds up to exactly the 2,551,620 total deal size, so nothing is lost or double counted.

## Two fields I stopped trusting

When I grouped the Probability column by sales stage, the averages made no sense. Won deals averaged 68% and Lost deals averaged 70%. Every stage sat between 68% and 71%. A probability field that gives Lost deals a higher average than Won deals isn't predicting anything.

The Weighted Forecast column is deal size times that probability, so it had the same flaw.

I kept both columns, renamed them "Reported Probability (Unreliable)" and "Reported Weighted Forecast (Unreliable)", and built a replacement. A new Stage Probability column maps each stage to a fixed value: Identified 10%, Validated 25%, Qualified 50%, Proposal 75%, Won 100%, Lost 0%. The corrected weighted forecast is deal size times that stage probability, written as a DAX measure so it recalculates with any filter.

The result is a weighted pipeline of about 902K against a total pipeline of 2.55M. Those stage percentages are my assumptions. A real company should replace them with its own historical conversion rates. I put a short note about all this on the dashboard's Data Quality page so a reader sees the issue instead of finding out later.

## The dashboard

There are six pages.

The Executive Summary has the total pipeline, weighted forecast, overdue value, win rate and data quality score, plus a slicer set for year, channel and country. Below that sit a funnel of open deals by stage and a Won versus Lost donut. I originally had Won and Lost as the last two steps of the funnel. That implies a conversion rate they don't represent, since a deal can be lost from any stage, so I split them.

The Forecast page shows confirmed versus potential value by month and by quarter, plus a matrix by sales channel. Confirmed is Won deals only (136,970). Potential is the four open stages (2,332,390). Lost deals, worth 82,260, are excluded from both.

Client Analysis shows the top 10 opportunities, pipeline by country and win rate. Country Analysis shows win rate by country and confirmed versus potential by country and quarter. Workload & Submissions has deal count and overdue count by owner, plus a detail list of open deals past their expected close date. The Data Quality page shows the reported versus stage-based probability comparison, the Next Steps breakdown and the data note.

Two sorting problems took longer than they should have. Months were sorting alphabetically, which put February and March 2015 in the wrong places. I fixed that with a numeric year-month sort key. Quarters had no year attached, so Q1 2015 and Q1 2016 were being merged into one bar. A combined quarter and year field with its own sort key fixed that.

## Checking the numbers

I checked the headline figures against the source file in Excel instead of trusting Power BI to check itself. Total deal size was 2,551,620 and the row count was 1,000, both matching. Stage counts were 355 Identified, 226 Validated, 188 Qualified, 145 Proposal, 53 Won and 33 Lost, which adds up to 1,000 and matches the funnel. Win rate by count, worked out by hand as 53 divided by 86, was 61.63%, matching the measure. Win rate by value comes out slightly higher at 62.48%, because larger deals were won a bit more often. I also spot-checked individual customers between the file and the report.

## What this doesn't do

Every customer in the dataset appears once, so there are no repeat clients. True client conversion rates and repeat-client analysis aren't possible, and I used deal-level win rate and country as the closest substitute. There's also no division field, so country stands in for it.

The overdue measure needs an honest explanation. My first version compared close dates to the latest reporting date (31 March 2015) and returned nothing, because no open deal had a close date that early. I anchored it to the latest close date in the data (24 September 2015) instead. That makes 913 of the 914 open deals overdue, which is a quirk of looking at a historical snapshot, not a real finding about the pipeline. The dashboard says "Pipeline snapshot as of 24 September 2015" so nobody reads it as live.

The monthly spread is an equal split. A day-weighted split is possible, but equal is simpler to explain, and I'd rather match what a client actually asks for.

## Using this on a real tender register

The same structure would work on an Excel register held in SharePoint or OneDrive. The bridge table is built from the start date, end date and value columns, so a client wouldn't need separate monthly forecast columns, and new rows would flow through on refresh.

My data had no blank or "TBC" dates, because I generated the dates myself. On a real register I'd flag those rows, keep them out of the bridge table, and show the value excluded from the forecast on a data quality page, so nobody assumes the totals are complete when they aren't.

## Key measures

```
Monthly Spread Value =
CALCULATE(
    SUM('Tender-Month Bridge'[Allocated Monthly Value]),
    USERELATIONSHIP('Date Table'[Date], 'Tender-Month Bridge'[Month Start Date])
)

Corrected Weighted Forecast =
SUMX('Dataset', 'Dataset'[Deal size] * 'Dataset'[Stage Probability])

Confirmed Spread Value =
CALCULATE([Monthly Spread Value], 'Dataset'[Sales stage] = "Won")

Potential Spread Value =
CALCULATE(
    [Monthly Spread Value],
    'Dataset'[Sales stage] IN {"Identified", "Validated", "Qualified", "Proposal"}
)

Win Rate (Count) =
DIVIDE(
    CALCULATE(COUNTROWS('Dataset'), 'Dataset'[Sales stage] = "Won"),
    CALCULATE(COUNTROWS('Dataset'), 'Dataset'[Sales stage] IN {"Won", "Lost"})
)

As Of Date =
CALCULATE(MAX('Dataset'[Close Date/Expected Close Date]), ALL('Dataset'))
```

Dataset source: Sales Pipeline dataset on Kaggle (ashishpandey5210). Project start and end dates were added by me and are synthetic.
