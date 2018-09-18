# splunk

##Some Splunk Query Syntax:

Query to find top to create a pie chart

sourcetype="access_combined_*" status!=200 "action=purchase" | top categoryId

Search by specific IP:

sourcetype="access_combined_*" status=200 action=purchase clientip=87.194.216.51 |stats count

Sory by each client id the products purchased

sourcetype="access_combined_*" status=200 action=purchase|stats count, dc(productId), values(productId) by clientip

Using Limit:

sourcetype="access_combined_*" status=200 action=purchase | top limit=1 clientip|table clientip,host

Same as below query but without renaming columns:

sourcetype="access_combined_*" status=200 action=purchase [search sourcetype=access* status=200 action=purchase | top limit=1 clientip|table clientip ]|stats count,dc(productId), values(productId) by clientip

Example to pipe and subquery [] and rename tables:

sourcetype="access_combined_*" status=200 action=purchase [search sourcetype=access* status=200 action=purchase | top limit=1 clientip|table clientip ]|stats count AS "Total Purch", dc(productId) AS "Total Prods", values(productId) AS "Prod IDs" by clientip | rename clientip as "the big customer"

status!=200 action=purchase | stats count by productId | sort by count desc


status!=200 action=purchase | stats count by clientip, productId | sort by count desc

big shopper query with charting:

sourcetype="access_combined_*" status=200|chart count as views count(eval(action="addtocart")) as addtocart count(eval(action="purchase")) as purchases by productName | rename productName as "Product Name", views as "Views", addtocard as "Adds to Cart", purchases as "Purchases"

overlay table:

sourcetype="access_combined_*" status=200|chart count as views count(eval(action="addtocart")) as addtocart count(eval(action="purchase")) as purchases by productName | eval viewstopurchases=(purchases/views)*100 | eval carttopurchases=(purchases/addtocart)*100| table productName views addtocart purchases viewstopurchases carttopurchases | rename productName as "Product Name", views as "Views", addtocart as "Adds to Cart", purchases as "Purchases"

remove nulls: 
sourcetype=access_* |timechart count(eval(action="purchase")) by productName usenull=f useother=f

have your results show up as a table:

index=main sourcetype="access_combined"|table *

or specifiy column name: 

index=main sourcetype="access_combined" | Table JSESSIONID

table but subtracting out columns:

index=main sourcetype="access_combined" | fields - index, sourcetype, source, date, _raw, lincecount, punct | table*

Top Query:

index=main sourcetype="access_combined" | top limit=20 uri_path

Investigate browsers top:

index=main sourcetype="access_combined" | eval browser=useragent|replace *Firefox* with Firefox, *Chrome* with Chrome, *MSIE* with IE, *Version*Safari with Safari, *Opera* with Opera |top limit=5 useother=t browser

Search by OS:

index=main sourcetype="access_combined" | eval os=useragent|replace *Macintosh* with Mac, *Windows* with Windows, *Linux* with Linux in os|top limit=5 useother=t os
