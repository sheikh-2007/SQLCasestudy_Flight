# ✈️ Airline Operations, Route Pricing & Passenger Analytics (SQL Case Study)

## 📌 Executive Summary
This case study analyzes a relational airline database (`Flights`, `Tickets`, `Airports`, `Airlines`, and `Passengers`) using **Microsoft SQL Server (T-SQL)**. The objective is to extract actionable commercial and operational insights across airport hub utilization, carrier market share, route pricing extremes, and passenger lifetime value (LTV).

---

## 🗄️ Database Schema Overview
The analysis queries across five relational tables:
* **`Flights`**: `FlightID`, `AirlineID`, `Origin` (FK to Airports), `Destination` (FK to Airports), `DepartureTime`, `ArrivalTime`
* **`Tickets`**: `TicketID`, `FlightID`, `PassengerID`, `Price`
* **`Airports`**: `AirportID`, `Name`
* **`Airlines`**: `AirlineID`, `Name`
* **`Passengers`**: `PassengerID`, `Name`, `FrequentFlyerStatus`

---

## 🛠️ Key SQL Skills Demonstrated
* **Complex Relational Joins:** Multi-table `INNER JOIN`s and dual-alias joins on the `Airports` table to resolve both Origin and Destination names simultaneously.
* **Window Functions & Partitioning:** Using `RANK() OVER (PARTITION BY ... ORDER BY ...)` to isolate top-performing airlines per hub, peak ticket prices per route, and top spenders per loyalty tier.
* **Common Table Expressions (CTEs):** Structuring multi-step aggregations for readability and performance.
* **Date/Time & Conditional Logic:** Calculating flight durations in minutes via `DATEDIFF` and categorizing haul lengths with `CASE` statements.

---

## 📊 Business Questions & SQL Solutions

### Q1. Identify the busiest airport by total departing flights
```sql
SELECT TOP 1 
    a.Name AS AirportName, 
    COUNT(*) AS TotalFlights
FROM Flights f
INNER JOIN Airports a ON f.Origin = a.AirportID
GROUP BY a.Name
ORDER BY TotalFlights DESC;
```

### Q2. Calculate total ticket volume sold per airline
```sql
SELECT 
    a.Name AS Airline,
    COUNT(*) AS TicketsSold
FROM Tickets t
INNER JOIN Flights f ON t.FlightID = f.FlightID
INNER JOIN Airlines a ON f.AirlineID = a.AirlineID
GROUP BY a.Name
ORDER BY TicketsSold DESC;
```

### Q3. Map all routes operated by 'IndiGo' with Origin & Destination airport names
```sql
SELECT 
    f.FlightID,
    ap_orig.Name AS OriginAirport,
    ap_dest.Name AS DestinationAirport
FROM Flights f
INNER JOIN Airlines a ON f.AirlineID = a.AirlineID
INNER JOIN Airports ap_orig ON f.Origin = ap_orig.AirportID
INNER JOIN Airports ap_dest ON f.Destination = ap_dest.AirportID
WHERE a.Name = 'IndiGo';
```

### Q4. Identify the dominant airline at each airport by departure volume
```sql
WITH CTE_FlightRank AS (
    SELECT 
        Origin, 
        AirlineID, 
        FlightCount,
        RANK() OVER (PARTITION BY Origin ORDER BY FlightCount DESC) AS rn
    FROM (
        SELECT Origin, AirlineID, COUNT(*) AS FlightCount
        FROM Flights
        GROUP BY Origin, AirlineID
    ) t
)
SELECT 
    a.Name AS AirportName, 
    al.Name AS AirlineName, 
    r.FlightCount
FROM CTE_FlightRank r
INNER JOIN Airports a ON r.Origin = a.AirportID
INNER JOIN Airlines al ON r.AirlineID = al.AirlineID
WHERE r.rn = 1;
```

### Q5. Categorize flights into Short (<2h), Medium (2–5h), or Long (>5h) haul
```sql
SELECT 
    FlightID,
    DepartureTime,
    ArrivalTime,
    DATEDIFF(MINUTE, DepartureTime, ArrivalTime) / 60.0 AS DurationHours,
    CASE
        WHEN DATEDIFF(MINUTE, DepartureTime, ArrivalTime) < 120 THEN 'Short (<2h)'
        WHEN DATEDIFF(MINUTE, DepartureTime, ArrivalTime) <= 300 THEN 'Medium (2-5h)'
        ELSE 'Long (>5h)'
    END AS FlightCategory
FROM Flights;
```

### Q6. Customer retention profile: First flight, Last flight, and total trips per passenger
```sql
WITH CTE_PassengerActivity AS (
    SELECT 
        t.PassengerID, 
        MIN(f.DepartureTime) AS FirstFlight,
        MAX(f.DepartureTime) AS LastFlight,
        COUNT(*) AS TotalFlights
    FROM Tickets t
    INNER JOIN Flights f ON t.FlightID = f.FlightID
    GROUP BY t.PassengerID
)
SELECT 
    p.Name AS PassengerName,
    cte.FirstFlight,
    cte.LastFlight,
    cte.TotalFlights
FROM CTE_PassengerActivity cte
INNER JOIN Passengers p ON cte.PassengerID = p.PassengerID
ORDER BY cte.TotalFlights DESC;
```

### Q7. Revenue analysis: Highest-priced ticket sold for each Origin-Destination route
```sql
WITH CTE_RouteTickets AS (
    SELECT 
        f.FlightID,
        f.Origin,
        f.Destination,
        t.TicketID,
        t.Price,
        RANK() OVER (PARTITION BY f.Origin, f.Destination ORDER BY t.Price DESC) AS rnk
    FROM Tickets t
    INNER JOIN Flights f ON t.FlightID = f.FlightID
)
SELECT 
    a1.Name AS Origin, 
    a2.Name AS Destination,
    rt.Price AS HighestTicketPrice,
    rt.TicketID
FROM CTE_RouteTickets rt
INNER JOIN Airports a1 ON rt.Origin = a1.AirportID
INNER JOIN Airports a2 ON rt.Destination = a2.AirportID
WHERE rt.rnk = 1;
```

### Q8. Customer segmentation: Highest-spending passenger within each Frequent Flyer tier
```sql
WITH CTE_Spending AS (
    SELECT 
        PassengerID, 
        Name, 
        FrequentFlyerStatus, 
        TotalSpent,
        RANK() OVER (PARTITION BY FrequentFlyerStatus ORDER BY TotalSpent DESC) AS rn
    FROM (
        SELECT 
            p.PassengerID, 
            p.Name, 
            p.FrequentFlyerStatus, 
            SUM(t.Price) AS TotalSpent
        FROM Passengers p
        INNER JOIN Tickets t ON p.PassengerID = t.PassengerID
        GROUP BY p.PassengerID, p.Name, p.FrequentFlyerStatus
    ) t
)
SELECT 
    Name AS PassengerName, 
    FrequentFlyerStatus, 
    TotalSpent
FROM CTE_Spending
WHERE rn = 1;
```
