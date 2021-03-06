# java4eurostat

[Java4eurostat](https://github.com/eurostat/java4eurostat) is a Java library for statistical data manipulation. It provides a number of functions to load statistical data into an 'hypercube' structure and index it for easy and fast in-memory computations. A number of specific functions are provided to easily access [Eurostat](http://ec.europa.eu/eurostat/) data.

## Quick start

Let's start with a simple dataset:

|country|gender|year|population|
|:--:|:--:|:--:| --:|
|Brasil|Male|2013|45.1|
|Brasil|Female|2013|48.3|
|Brasil|Total|2013|93.4|
|Brasil|Male|2014|46.2|
|Brasil|Female|2014|47.7|
|Brasil|Total|2014|93.9|
|Japan|Male|2013|145.1|
|Japan|Female|2013|148.3|
|Japan|Total|2013|293.4|
|Japan|Male|2014|146.2|
|Japan|Female|2014|147.7|
|Japan|Total|2014|293.9|

stored as a CSV file `example.csv`:

```
country,gender,year,population
Brasil,Male,2013,45.1
Brasil,Female,2013,48.3
Brasil,Total,2013,93.4
Brasil,Male,2014,46.2
Brasil,Female,2014,47.7
Brasil,Total,2014,93.9
Japan,Male,2013,145.1
Japan,Female,2013,148.3
Japan,Total,2013,293.4
Japan,Male,2014,146.2
Japan,Female,2014,147.7
Japan,Total,2014,293.9
```

This file can be loaded into an hypercube structure with:

```java
StatsHypercube hc = CSV.load("example.csv", "population");
```

Information on the hypercube structure is shown with ```hc.printInfo();```, which returns:

```
Information: 12 value(s) with 3 dimension(s).
   Dimension: gender (3 dimension values)
      Female
      Male
      Total
   Dimension: year (2 dimension values)
      2013
      2014
   Dimension: country (2 dimension values)
      Brasil
      Japan
```

Several input formats are supported. For example, [Eurostat](http://ec.europa.eu/eurostat/) data can be loaded directly from the web. For that, only the database code given in [Eurostat databases catalog](http://ec.europa.eu/eurostat/data/database) is required. For example, the database on *HICP - Country weights* (code *prc_hicp_cow*) can be downloaded and loaded simply with:

```java
StatsHypercube hc2 = EurobaseIO.getData("prc_hicp_cow");
```

The structure returned with a ```hc2.printInfo();``` is:

```
Information: 2001 value(s) with 3 dimension(s).
   Dimension: time (21 dimension values)
      1996
      1997
      1998
      ...
   Dimension: geo (35 dimension values)
      AT
      BE
      BG
      ...
   Dimension: statinfo (6 dimension values)
      COWEA
      COWEA18
      COWEA19
      ...
```

Once loaded, data can be filtered/selected. For example, ```hc.selectDimValueEqualTo("country","Brasil")``` selects data for Brasil and ```hc.selectValueGreaterThan(147)``` selects data with values greater than 147. Selection criteria can be combined in cascade like ```hc.selectDimValueEqualTo("country","Brasil").selectDimValueGreaterThan("year",2012)``` for the selection of Brasil data after 2012. Logical operators 'AND', 'OR' and 'NOT' can also be used to build more complex selection criteria. Totally generic selection criteria can be specified such as:

```java
hc.select(new Criteria(){
	@Override
	public boolean keep(Stat stat) {
		return stat.dims.get("country").contains("r") && Math.sqrt(stat.value)>7;
	}
});
```

which selects all statistics with country names containing a "r" character, and whose root square value is greater than 7.

A single value can be retrieved with for example ```hc.selectDimValueEqualTo("country", "Japan", "gender", "Total", "year", "2014").stats.iterator().next().value``` but the fastest way to retrieve a value and scan a dataset is to use an index with:

```java
StatsIndex index = new StatsIndex(hc,"gender","year","country");
```
This index is a tree structure based on the dimension values. This structure can be displayed with ```index.print();```:

```
Total
   2014
      Brasil -> 93.9
      Japan -> 293.9
   2013
      Brasil -> 93.4
      Japan -> 293.4
Male
   2014
      Brasil -> 46.2
      Japan -> 146.2
   2013
      Brasil -> 45.1
      Japan -> 145.1
Female
   2014
      Brasil -> 47.7
      Japan -> 147.7
   2013
      Brasil -> 48.3
      Japan -> 148.3
```

A statistical value is accessed quickly from the index and its dimension values: ```double value = index.getSingleValue("Total","2014","Japan");```. Scanning a full dataset across its dimensions is very fast with:

```java
for(String gender : index.getKeys())
	for(String year : index.getKeys(gender))
		for(String country : index.getKeys(gender,year)) {
			System.out.println(gender +" "+year+" "+country);
			System.out.println(index.getSingleValue(gender,year,country));
		}
```

## More information

### Setup

To quickly setup a development environment, see [these instructions](https://eurostat.github.io/README/howto/java_eclipse_maven_git_quick_guide).

Download and install [Java4eurostat](https://github.com/eurostat/java4eurostat) with:

```
git clone https://github.com/eurostat/java4eurostat.git
cd java4eurostat
mvn clean install
```

and then use it in your Java project as a dependency by adding it to the *pom.xml* file:

```
<dependencies>
	...
	<dependency>
		<groupId>eu.europa.ec.eurostat</groupId>
		<artifactId>java4eurostat</artifactId>
		<version>1.3</version>
	</dependency>
</dependencies>
```

### Input data
#### CSV
```java
//load
StatsHypercube hc = CSV.load("example.csv", "population");
//save
CSV.save(hc, "C:\datafolder\", "dataset.csv");
```
#### JSON-stat
See [here](https://json-stat.org/) for more information on this format.
```java
//load
String jsonStatString = '{"version":"2.0", "class":"dataset", "label":"Population data", "source":"", "id":[...], "size":[...], "dimension":{...}, "value":[...]}';
StatsHypercube hc = JSONStat.load(jsonStatString);
//save
//  not implemented (yet)
```
#### Eurostat data

The class ```EurobaseIO``` provides several functions to handle Eurostat data. For example: ```StatsHypercube hc = EurobaseIO.getData("prc_hicp_cow");``` loads the database *prc_hicp_cow*. Selection parameters may also be specified: ```getData("prc_hicp_cow", "geo", "EU", "geo", "EA", "time", "2016")``` returns loads database *prc_hicp_cow* figures for 2016, for both *EU* and *EA*. Additionnaly, ```getData("prc_hicp_cow", "lastTimePeriod", "4")``` return the figures for the 4 last time periods, while ```getData("prc_hicp_cow", "sinceTimePeriod", "2005")``` returns all figures since 2005.

Eurostat TSV files can be downloaded manually from [the bulk download facility](http://ec.europa.eu/eurostat/data/bulkdownload) or using:

```java
//download from Eurostat bulk download facility
EurobaseIO.getDataBulkDownload("eurobase_code","/home/datafolder/");
//load
StatsHypercube hc = EurostatTSV.load("/home/datafolder/eurobase_code.tsv");
//save
//  not implemented (yet)
```

The last publication date of a database can be retrieved with ```getUpdateDate```: For example, ```EurobaseIO.getUpdateDate("prc_hicp_cow");``` returns the last publication date of the database with code *prc_hicp_cow*.

In case of regular use of some Eurostat databases as TSV files, these files can be downloaded and updated only when new data is published. For example:

```java
EurobaseIO.update("C:/my_data_folder/", "my_database_code1", "my_database_code2", "my_database_code3", ...);
```

retrieves new files *my_database_code1.tsv*, *my_database_code2.tsv* and *my_database_code3.tsv* only when they has been updated. This function creates a file ```update.txt``` in ```C:/my_data_folder/``` folder, which gives the last update dates of the files. 

Code list dictionnaries are loaded with for example ```EurobaseIO.getDictionnary("geo")``` which retrieve the dictionnary of geographical locations (code *geo*). ```EurobaseIO.getDictionnary("geo").get("IT")``` returns "Italy". Last update dates are retreved with for example ```getDictionnaryUpdateDate("geo")```.

#### Filtering on loading
To ensure an efficient usage of memory, a selection criteria can be specified when loading from a data source. For example, ```StatsHypercube hc = EurobaseIO.getData("prc_hicp_cow", new DimValueEqualTo("geo","BG"))``` loads only data for country *BG*.

### Basic data structures

The base classes are ```Stat``` and  ```StatsHypercube```. A ```Stat``` object represents a statistical values, element of a ```StatsHypercube``` object.

A ```Stat``` object is characterised by its value (of course) and a dictionnary of pairs *(dimension label, dimension value)*, which represents its coordinates within the hypercube. [Flags](http://ec.europa.eu/eurostat/data/database/information) can also be attached to a statistical value. The class ```StatsHypercube``` is simply characterised by its collection of ```Stat``` elements and dimension names.

[TODO: describe HierarchicalCode]

### Data access

Data of a hypercube are accessed using either the ```StatsHypercube.select()``` method or a ```StatsIndex``` object. Access with a ```StatsIndex``` is faster, but requires the construction of an index object, which can be resource consumming.

Basic operations based on selection and indexing are presented in the quick start section above.

[TODO: extend description.]

### Time series analysis

The class ```TimeSeriesUtil``` provides several function for time series analysis such as the computation of moving averages, gap analysis and outlier values detection.

[TODO: extend description.]

### Further documentation

See [the documentation](https://eurostat.github.io/GridMaker/apidocs/) for more information.

## Support and contribution

Feel free to [ask support](https://github.com/eurostat/java4eurostat/issues/new), fork the project or simply star it (it's always a pleasure).
