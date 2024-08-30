# Notes on Environmental Data Management 

More and more open-data available. Unthinkable 10 years ago. More and more NWPs offer open-data. Higher resolution, global and local area models. Enormous amounts of data, get harder and harder to access.

Open-Meteo sets out to change it. Fully open-source. Redistribute open-data environmental datasets with simple and fast APIs. Open-access for non-commercial use and research. Integrates models from most important NWPs for Europe, North America and Asia.

The following describes how open-meteo is handling large amounts of environmental data, how it works and the direction for future developments for open-meteo as well as other users of environmental data.

## Problem space
working with environmental data like weather forecast model output, reanalysis or satellite data in practice is challenging for end users.

1. Data is big and getting bigger. But only need small portion of data -> have to download hundrets of GB of GRIB files. Huge CPU decoding time, throw away 99% of downloaded and processed data. A lot of effort that is not resource intensive, but also requires a lot of know how. Downloading like ERA5 even for just one location, may take 3-6 months.
2. Difficult to use data. GRIB files, projections, aggregations, jumps from 1 to 3 to 6h data. Correct processing of solar radiation in order to calculate common variables like evapo reference transpiration ET0 or direct normalized irradiation (DNI) is immensely difficult for novice users.
3. So many file formats and standards from various national weather services. No generic solution possible. Getting to know open data from each NWP is time consuming. Hard to discover which data is available. No central catalogue.
4. Complicated integrations. To effectively get to a point where a user can "work" with weather data, it needs countless of libraries, tinkering to download and store data on a workstation, working with projections to find coordinates and work around problems like you cannot fit a year of ERA5-Land temperature data in memory.


## The current Open-Meteo approach
Open-Meteo is only a small step, but makes accessing data more transparent and faster. However, it does not yet solve all issues. Currently, Open-Meteo offers simple HTTP APIs to retrieve a small subset of weather data for individual locations.

Open-Meteo is mostly focused on returning time-series data for coordinates. This is the exact opposite of how environmental is usually stored and distributed. 

Weather models updates are continuously updated into a time-series database, which offers the convenient feature that users cannot only get the latest model forecast, but seamlessly get a time-series from the past model runs. This enables that users can get a weather forecast from the current day which may already incorporate 0z, 6z and 12z, but also get data from past weeks, months and even years.

The database is not limited to weather forecast models, but also includes reanalysis data like ERA5 and ERA5-Land. 80+ years of hourly for a single location data can be accessed within less than 100 milliseconds! This is only possible by storing data efficiently for this use-case and optimizing for performance.

What users can access:

1. Weather forecasts for individual coordinates with integration of local area models for short term forecasting and global models for medium-range forecasting
2. Past weather from high-resolution weather models (archives start ~2018) and reanalysis from ERA5 (1940 onwards with daily updates)
3. Access forecasts with lead-time offsets to optimize for one or two day-ahead forecasting as well as generate validations showing the skill of individual forecast days.
4. Access a large range of weather parameters including model and pressure level variables. Some parameters like Direct Normal Irradiance (DNI) or Reference Evapotranspiration (ET₀) are calculated on demand.
5. Access to individual ensemble member forecasts for probabilistic forecasting. Around 30 to 50 members are available for each model. 
6. Additional datasets like Air Quality based on Copernicus CAMS (EU + Global), Climate models (CMIPS HighRes), various ocean wave models, flood models (GloFAS)

Access is not limited to single coordinates. Multiple coordinates can be requested in one API call as well. Bounding boxes are supported as well, but limited to smaller areas. The bounding box feature is not documented yet.

Currently it is not possible to access forecasts for larger areas (large country / continent) or the entire globe. This is the worst case access pattern for the underlying database. User who want to generate maps, are asked to use GRIB files directly. However, support is technically possible, but requires some development to make it work.


## Storing data

GRIB files work well to store a large field for a single timesteps of a single variable. Compression is reasonable good. To read a single location or small area, the entire field needs to be decoded. If multiple GRIB messages are concatenated in a single file, this requires an index file to access an individual parameters, complicating it further to access GRIB files on demand.

An API serving weather data needs an underlying database which is specifically optimized to ingest large amounts of data and read small portions randomly preferable as a time-series.

The solution is relatively simple to structure weather data in multidimensional arrays, transpose for fast time access and store it on disk in formats like NetCDF, Zarr or HDF5. Data needs to be chunked (e.g. 50 gridpoints x 50 timesteps) and compressed using various formats like gzip, zstd, blosc, lz4 and others.

A 0.25° model with 80 forecast steps every 3 hours has the dimensions of [1440; 720; 80]. Instead of storing each run in its own file, a continuous time-series can be formed by creating one file for every calender week and merge data into existing weekly files, a continuous time-series forms automatically.

![Database run overlap](images/database_run_overlap.png)

By using simple files, data can then be arranged like this:
- `/temperature_2m/`
- `/temperature_2m/2024_week01.nc`
- `/temperature_2m/2024_week02.nc`
- `/temperature_2m/2024_week03.nc`
- `/precipitation/`
- `/precipitation/2024_week01.nc`
- `/precipitation/2024_week02.nc`
- `/precipitation/2024_week03.nc`

To return a forecast for the next 7 days, that starts at local-time midnight, simply open the corresponding weekly files and read data the correct days. Data for the past 10 weeks? Open the last 10 weeks files. Store all 51 ensemble members? Simply add another dimension [1440; 720; 51; 80].

To update the database the workflow looks like:
1. Download GRIB files of latest run
2. Transpose spatial oriented data to temporal access
3. Read existing weekly files
4. Merge new updated data
5. Overwrite weekly files
6. Repeat with next model run

In a very simplified way, this is how Open-Meteo stores weather model updates as a continuous time-series. In practice, individual files are not fixed to individual calendar weeks and updating files is quite a challenge considering that some model updates are hundreds of GB in size.

Existing file formats like NetCDF, HDF5 or Zarr, 


All data needs to be downloaded and stored differently for fast access. The exciting formats like GRIB do not offer each access. No database, simple files. 

Work with big gridded data files. Optimised for exactly this use case. Does not support non-gridded data like station measurements.

Transposed to time-series data. Access not only the last model runs, but maintain a time-series of data.

Analysed many different file formats and compressions schemas, decided to built its own. Used since 2 years. Would to the same.

## Open-Meteo custom file format

File = compressed multi dimensional array. Chunked data format. Enable reads of small parts without reading whole file. Similar to HDF5 or Zarr. 

Access via S3 protocol. Independent from cloud provider. Can work on multiple clouds. Technically any HTTP server works. PoC S3 sponsorship. 

Data organization in directories.

High compression with fixed precision, delta encoding (spatial and temporal correlation), integer encoding. Option for lossless float compression, but larger files. Vector instructions, modern CPUs.

Two ways of storage
1. Transposed data format for fast time-series access. Rewrites parts of each database with each update.
2. GRIB replacement, one file per timestep, per field. Multiple members can be in same file (=dimension), Issue: lots of small files

## Processing data from NWPS

Data conversion routines downloading GRIB/NetCDF files from various NWP. Also includes ERA5. Convert to one standardized database / naming.
High compression speed / fast updates. Updates files, upload to S3. Additional data sources can be integrated.

Chunk wise data transpose to use less than 500GB memory ;-)

Statistics:
- GRIB data processed per day
- Compression ratio ~5 less than GRIB + keep continuous time-series of model data. Example ERA5
- Number of api call per day
- Database size



## HTTP Rest APIs
Provide easy and fast access to small portions of data. Mostly single coordinates, but bounding boxes are supported.

Can use files on S3, with large local cache as SSD.

Open-Meteo uses SSD only nodes for forecast, hybrid storage for archives. Uses simple sync protocol based on S3-like.

Scalability: just add new nodes

Docker instances available

## Client libraries
Currently only libraries to interact with API. 

Work in progress:
- Make file format more generic
- Client libraries for python, etc

Direct access to files on S3 storage. Local cache. Enable users to run analysis without worrying about data transfer, etc. Strict backwards compatibility.

Other languages like Javascript, WASM, R, Julia, Rust, Swift, etc

Make file format usable in different sectors as well. Areas to improve global metadata schema and standards.

Users still have to option to download, store and archive the whole file.
