# ETL Is Dead, Long Live Streams: real-time streams with Apache Kafka
> [Original Video](https://www.youtube.com/watch?v=I32hmY4diFY&list=WL&index=42)

## Recent data trends
- single-server databases are replaced by a myraid of distributed data platforms that operate at company-wide scale.
- many more types of data sources beyond transactional data - logs, sensors, metrics, ...
- stream data is increasingly ubiquitous need for faster processing than daily.

## ETL drawbacks
- need a global schema for uniform warehousing.
- data cleansing and curation is manual and fundamental error-prone
- operational cost is high
- tools were built to narrowly ficus on connecting databases and the data warehouse in a batch fashion
- most of time are batch operation, not suitable for real-time task

## Requirements of Modern Streamming System
- high-volumn and high-diversity data
- real-time from the grounds up (event centric thinking)
