## Cleaning Data In SQL Queries
In this project we will perform data cleaning using SQL queries.

Here is a glance at the table we are working with:

````sql
Select Top(3) *
From housing_data
````
|UniqueID| ParcelID |LandUse|PropertyAddress |SaleDate |SalePrice|             LegalReference|SoldAsVacant|OwnerName|OwnerAddress|Acreage|TaxDistrict|LandValue| BuildingValue|TotalValue | YearBuilt  | Bedrooms | FullBath |HalfBath|
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
|2045|    007 00 0 125.00   | SINGLE FAMILY |  1808  FOX CHASE DR, GOODLETTSVILLE  | 2013-04-09 00:00:00.0000000 |240000.00 | 20130412-0036474    |   No   |   FRAZIER, CYRENTHA LYNETTE   |  1808  FOX CHASE DR, GOODLETTSVILLE, TN      |    2.3       |   GENERAL SERVICES DISTRICT   | 50000 |     168200   |  235700   |   1986    |    3    |   3   |   0|
|16918 |      007 00 0 130.00     |                               SINGLE FAMILY     |                                 1832  FOX CHASE DR, GOODLETTSVILLE      |           2014-06-10 00:00:00.0000000 |366000.00     |        20140619-0053768      |                             No |   BONER, CHARLES & LESLIE      |  1832  FOX CHASE DR, GOODLETTSVILLE, TN|   3.5    |  GENERAL SERVICES DISTRICT   |  50000    |   264100    | 319000  |    1998   |    3     | 3    |   2|
|54582|       007 00 0 138.00        |                            SINGLE FAMILY     |                                 1864 FOX CHASE  DR, GOODLETTSVILLE      |           2016-09-26 00:00:00.0000000| 435000.00    |         20160927-0101718       |                            No                           |                      WILSON, JAMES E. & JOANNE           |                                                                 1864  FOX CHASE DR, GOODLETTSVILLE, TN|             2.9                     |                           GENERAL SERVICES DISTRICT  |                        50000   |    216200       |                                      298000   |   1987  |      4                   |                               3      |                                            0|

### 1. Changing the date format

Let us first look at how the date is formatted
````sql
Select Top(10) SaleDate
From housing_data;
````
|SaleDate|
|----|
|2013-04-09 00:00:00.0000000|
|2014-06-10 00:00:00.0000000|
|2016-09-26 00:00:00.0000000|
|2016-01-29 00:00:00.0000000|
|2014-10-10 00:00:00.0000000|
|2014-07-16 00:00:00.0000000|
|2014-08-28 00:00:00.0000000|
|2016-09-27 00:00:00.0000000|
|2015-08-14 00:00:00.0000000|
|2014-08-29 00:00:00.0000000|

The SaleDate attribute is in DateTime format however, the time is not useful here. Therefore, we will convert it to just the date.

````sql
Select Top(10) CONVERT(Date, SaleDate) as SaleDate
From housing_data;
````
|SaleDate|
|----|
|2013-04-09|
|2014-06-10|
|2016-09-26|
|2016-01-29|
|2014-10-10|
|2014-07-16|
|2014-08-28|
|2016-09-27|
|2015-08-14|
|2014-08-29|

Make the change in the table:

````sql
Update housing_data
SET SaleDate = CONVERT(Date,SaleDate)
````
### 2.  Populate  Missing Property Address

We have some rows in our data set with missing Property Address.

````sql
Select count(*) as missing_addresses
From housing_data
Where PropertyAddress is null;
````
|missing_addresses|
|----|
|406|

We find in our table that there are entries with the same ParcelID.  These rows should have the same property address. We check if we can fill in the missing addresses using other rows with matching ParcelID.

````sql
Select
  a.ParcelID, 
  a.PropertyAddress,
  b.ParcelID, 
  b.PropertyAddress,
  -- if a.PropertyAddress is null then return b.PropertyAddress
  ISNULL(a.PropertyAddress,b.PropertyAddress) as check
From housing_data a
JOIN housing_data b
	on a.ParcelID = b.ParcelID
	AND a.[UniqueID] <> b.[UniqueID]
Where a.PropertyAddress is null;
````
|a.ParcelID|a.PropertyAddress|b.ParcelID|b.PropertyAddress|check|
|-----|---|-----|-----|-----|
|042 13 0 075.00|NULL|042 13 0 075.00|222  FOXBORO DR, MADISON|222  FOXBORO DR, MADISON|
|043 04 0 014.00|NULL|043 04 0 014.00|112  HILLER DR, OLD HICKORY|112  HILLER DR, OLD HICKORY|
|043 09 0 074.00|NULL|043 09 0 074.00|213 B  LOVELL ST, MADISON|213 B  LOVELL ST, MADISON|
|043 13 0 308.00|NULL|043 13 0 308.00|224  HICKORY ST, MADISON|224  HICKORY ST, MADISON|
|044 05 0 135.00|NULL|044 05 0 135.00|202  KEETON AVE, OLD HICKORY|202  KEETON AVE, OLD HICKORY|
|025 07 0 031.00|NULL|025 07 0 031.00|410  ROSEHILL CT, GOODLETTSVIL|410  ROSEHILL CT, GOODLETTSVIL|
|026 01 0 069.00|NULL|026 01 0 069.00|141  TWO MILE PIKE, GOODLETTSV|141  TWO MILE PIKE, GOODLETTSV|
|026 05 0 017.00|NULL|026 05 0 017.00|208  EAST AVE, GOODLETTSVILLE|208  EAST AVE, GOODLETTSVILLE|
|026 06 0A 038.00|NULL|026 06 0A 038.00|109  CANTON CT, GOODLETTSVILLE|109  CANTON CT, GOODLETTSVILLE|
|033 06 0 041.00|NULL|033 06 0 041.00|1129  CAMPBELL RD, GOODLETTSVI|1129  CAMPBELL RD, GOODLETTSVI|

Indeed, we find that we can fill in some of the missing addresses using other rows with same ParcelID.  Now, we make the update in our table

````sql
Update a
SET PropertyAddress = ISNULL(a.PropertyAddress,b.PropertyAddress)
From housing_data a
JOIN housing_data b
	on a.ParcelID = b.ParcelID
	AND a.[UniqueID] <> b.[UniqueID]
Where a.PropertyAddress is null;
````

### 3. Break out the address into individual columns (address, city)
We notice that the PropertyAddress columns contains the address and the city. We should separate that into individual columns. We will use the substring() function.
````sql
SELECT Top(10)
	propertyaddress,
	-- the characters before the comma make up the address
	SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1 )  as prop_address,
	SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) + 1 , LEN(PropertyAddress)) as prop_city
From housing_data
````
|PropertyAddress|prop_address|prop_city|
|-----|-----|-----|
|1808  FOX CHASE DR, GOODLETTSVILLE|1808  FOX CHASE DR| GOODLETTSVILLE|
|1832  FOX CHASE DR, GOODLETTSVILLE|1832  FOX CHASE DR| GOODLETTSVILLE|
|1864 FOX CHASE  DR, GOODLETTSVILLE|1864 FOX CHASE  DR| GOODLETTSVILLE|
|1853  FOX CHASE DR, GOODLETTSVILLE|1853  FOX CHASE DR| GOODLETTSVILLE|
|1829  FOX CHASE DR, GOODLETTSVILLE|1829  FOX CHASE DR| GOODLETTSVILLE|
|1821  FOX CHASE DR, GOODLETTSVILLE|1821  FOX CHASE DR| GOODLETTSVILLE|
|2005  SADIE LN, GOODLETTSVILLE|2005  SADIE LN| GOODLETTSVILLE|
|1917 GRACELAND  DR, GOODLETTSVILLE|1917 GRACELAND  DR| GOODLETTSVILLE|
|1428  SPRINGFIELD HWY, GOODLETTSVILLE|1428  SPRINGFIELD HWY| GOODLETTSVILLE|
|1420  SPRINGFIELD HWY, GOODLETTSVILLE|1420  SPRINGFIELD HWY| GOODLETTSVILLE|

Update the table.

````sql
-- new column for address
ALTER TABLE housing_data
Add Prop_Address Nvarchar(255);

Update housing_data
SET Prop_Address = SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1 )

-- new column for city
ALTER TABLE housing_data
Add Prop_City Nvarchar(255);

Update housing_data
SET Prop_City = SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) + 1 , LEN(PropertyAddress))
````
We also need to split the OwnerAddress column into separate columns for address, city and state. We will use PARSENAME() function this time.
### 4. Make SoldAsVacant column uniform
We notice some inconsistencies in the  SoldAsVacant column
````sql
Select Distinct(SoldAsVacant), count(SoldAsVacant)
From housing_data
group by SoldAsVacant
order by 2;
````

|SoldAsVacant|count(SoldAsVacant)|
|----------| -----------|
|Y      |                                            713|
|N    |                                             5466|
|Yes |                                               62630|
|No|                                                 698964|

We should change the Y into Yes and the N into No.

````sql
Update housing_data
SET SoldAsVacant = CASE When SoldAsVacant = 'Y' THEN 'Yes'
	   When SoldAsVacant = 'N' THEN 'No'
	   ELSE SoldAsVacant
	   END;
````
### 5. Checking for Duplicates
````sql
WITH RowNumCTE AS(
Select *,
	ROW_NUMBER() OVER (
	PARTITION BY 
		ParcelID,
		PropertyAddress,
		SalePrice,
		SaleDate,
		LegalReference
	ORDER BY
		UniqueID) row_num
From housing_data
--order by ParcelID
)
Select *
From RowNumCTE
Where row_num > 1
Order by PropertyAddress
````
|UniqueID|ParcelID|Prop_Address|Prop_city|SaleDate|LegalReference|
|--------|----------|-------------|--------------|-------------|---------------|
|2979|118 01 0Q 106.00|  12TH AVE S| NASHVILLE|2013-05-28|20130603-0055710|
|2979|118 01 0Q 106.00|  12TH AVE S| NASHVILLE|2013-05-28|20130603-0055710|
|2868|104 11 0U 003.00|  ACKLEN AVE| NASHVILLE|2013-05-30|20130531-0055044|
|2868|104 11 0U 003.00|  ACKLEN AVE| NASHVILLE|2013-05-30|20130531-0055044|
|9095|063 12 0 063.00|  HADLEYS BEND BLVD| OLD HICKORY|2013-10-31|20131101-0113875|
|9095|063 12 0 063.00|  HADLEYS BEND BLVD| OLD HICKORY|2013-10-31|20131101-0113875|
|1894|131 03 1G 002.00|  LONE OAK RD| NASHVILLE|2013-04-29|20130430-0042844|
|1894|131 03 1G 002.00|  LONE OAK RD| NASHVILLE|2013-04-29|20130430-0042844|
|8995|072 07 0 089.00|  LOVE JOY CT| NASHVILLE|2013-10-07|20131008-0105750|
|8995|072 07 0 089.00|  LOVE JOY CT| NASHVILLE|2013-10-07|20131008-0105750|
|4063|104 06 0Q 001.00|  MARLBOROUGH AVE| NASHVILLE|2013-06-25|20130711-0071776|
|4063|104 06 0Q 001.00|  MARLBOROUGH AVE| NASHVILLE|2013-06-25|20130711-0071776|
|5062|163 07 0A 904.00|  WILD OAKS CT| ANTIOCH|2013-06-28|20130702-0068072|

At this point we should start normalizing the data as we did in this project: [link](https://github.com/Nilesh-Ramdany/Data_analysis_portfolio/blob/main/Data_Normalization_SQL.md)
