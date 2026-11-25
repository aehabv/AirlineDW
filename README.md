# Airline data warehouse

![Oracle SQL](https://img.shields.io/badge/Oracle%20SQL-F80000?style=flat&logo=oracle&logoColor=white)
![Data Vault](https://img.shields.io/badge/Data%20Vault-2C3E50?style=flat)
![License](https://img.shields.io/badge/License-CC%20BY--NC%204.0-0a2c46?style=flat)

An airline runs on bookings, flights, loyalty points, hotel stays and complaints, and every one of those lives in a different system. This project designs a data warehouse that pulls them together so the business can ask questions across all of them, like "how full were our planes in January?" or "which frequent flyers spend the most nights in partner hotels?"

It's a design project: the model, the assumptions and the queries. There's no data in the repo. (Funny enough, I now work on an air cargo data platform for real.)

## The business processes

I started from what the airline actually does and wrote down the grain of each process, meaning what one row represents:

| Business process | One row is… | Fact table | Dimensions |
|---|---|---|---|
| Booking | a ticket sold | Booking | Date, Passenger, Flight, Agency, Website, Promotion |
| Cancelling | a ticket sold | Booking | Date, Passenger, Flight |
| Flight, landing/take-off, transit | a flight | Flight | Date, Passenger, Airplane, Pilot, Crew, Flight, Route |
| Frequent flyer, upgrading, promotions, redeeming points | a loyalty transaction | Loyalty | Date, Flight, Frequent flyer, Points redeeming key, Passenger |
| Hotel booking | a stay | Staying | Date, Flight, Frequent flyer, Hotel, Passenger |
| Check-in, inquiries | an inquiry | Factless | Date, Passenger, Inquiry, Flight, Staff |
| Car rental | a rental | Factless | Date, Passenger, Flight |
| Accidents | an incident | Factless | Date, Flight |
| Maintenance | an airplane | Factless | Airplane |

"Factless" tables record that something happened, such as an inquiry or an accident, without a number to add up.

The full list of assumptions is in [`Business Processes and Assumptions.pdf`](Business%20Processes%20and%20Assumptions.pdf). A few that shape the model: frequent flyers earn points per dollar depending on fare basis, there are five membership tiers (Blue to Titanium), tickets come from either an agency or the website with different fees, and every seat has a ticket.

## Two layers

**A Data Vault for integration.** Hubs hold the core business keys (passengers, flights, airplanes), links hold the relationships between them (bookings, upgrades, cancellations), and satellites hold the details and their history. I picked Data Vault because new source systems can be added without reworking the model, it keeps the raw history, and it loads in parallel, which matters for an industry that never stops.

![Data Vault model](AIRLINE_SYSTEM_DV.png)

**A dimensional model for reporting.** Hubs and links are awkward to query, so a star schema sits on top for analysis: six fact tables (booking, flight, total flight, loyalty, staying and report inquiry) around 17 shared dimensions. The total flight fact rolls each flight up to revenue, cost, discounts, fees, tickets sold and profit.

![Facts and dimensions](AIRLINE_D&F.png)

The index and partitioning plan is in [`Indexes_and_Partitions.pdf`](Indexes_and_Partitions.pdf).

## Example questions

[`SQL QUERIES.txt`](SQL%20QUERIES.txt) has ten Oracle queries against the dimensional layer, including:

- total profit, cost and tickets sold in January
- seat utilisation (tickets sold divided by seats available)
- tickets booked through agencies rather than the website
- frequent flyers per membership tier
- inquiries by time (before, during or after the flight)
- hotel nights per frequent flyer, and the hotels with the most nights in January

## Caveats

- **No data, so nothing has been run end to end.** The queries show how the model would be used, but I can't show their output.
- **A couple of typos in the queries.** `PASSANGER` and `INTQUIRY_TIME` would fail against tables spelled the way the diagram spells them.
- **Some filters are shortcuts.** "January" is written as `date_key < 20210201`, which only works if the data starts on 1 January 2021, and "booked through an agency" is taken to mean the website key is empty.

## What I'd like to do next

Generate a realistic synthetic dataset, build both layers in a real database, and run the queries for real.
