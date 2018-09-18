# setup Docker
1) login to portal.azure.com
2) Create a resource group: ie: table5ayrg
3) Add a vm CoreOS Linux (Stable) to resource group.  Set a standard VM login (ie ops), and password.
4) Add a network security group to your resource group.  ie: table5ayvm-nsg
5) inbound security rule, Port_8000, port 800, any allow
6) inbound security rule, SSH, port 22, any, allow
7) Choose the VM, select connect, and copy SSH command 
8) open Azure cloud shell, now paste SSH command ie: ssh ops@_____ where blank is the IP of the VM
9) now grab the splunk docker container: sudo docker pull splunk/splunk/
10) sudo docker run -v /home/ops:/data -p 8000:8000 -d -e SPLUNK_START_ARGS="--accept-license --seed-password "standard password" splunk/splunk
11) we just mapped /home/ops to /data for splunk, -e is for daemon, and "standard password" is the admin password for this splunk instance.
12) sudo docker ps to list containers
13) now logon to splunk instance at your above VM ip in a web browser , ie 157.56.176.102:8000
14) login to splunk
15) upload test dataset: tutorial_data.zip
16) OpsDataGen.spl is the custom app to install.  I think this relates to log4j logging for JS.  Log4net would be C#.
17) prices.csv allows you to add price data to existing dataset
18) when done, stop your VM in azure.

To restart your docker image, list out dockers:
sudo docker ps -a
sudo docker start <container id>

To clean up dockers
sudo docker ps -a
sudo docker rm -f <container id>

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

Check referral domains, and rank + - ascending descending:
index=main sourcetype=access_combined|stats dc(clientip) as Referals by referer_domain|sort - Referals

index=main sourcetype=access_combined uri_path="/addItem" OR uri_path="/checkout" | chart count(eval(like(status, "2%"))) as Success, count(eval(like(status,"4%") Or like(status,"5%"))) as Error by uri_path|addcoltotals label=Total labelfield=uri_path

Time Chart

index=main sourcetype="access_combined" | timechart span=6h avg(response) AS avgResp | eval avgResp=round(avgResp/1000,2)

Dedup!!

index=main sourcetype="access_combined" status=200 uri_path="/viewItem" OR uri_path="/addItem" |dedup cookie uri_path item | top item

rank descending 

index=main sourcetype="access_combined" status=200 uri_path="/viewItem" OR uri_path="/addItem" |dedup cookie uri_path item | chart count(eval(uri_path="/viewItem")) as view, count(eval(uri_path="/addItem")) as add by item|sort - view| head 10

calculations in table and using head

index=main sourcetype="access_combined" status=200 uri_path="/viewItem" OR uri_path="/addItem" |dedup cookie uri_path item | chart count(eval(uri_path="/viewItem")) as view, count(eval(uri_path="/addItem")) as add by item|sort - view| head 10|eval cart_conversion=round(add/view*100)."%"

ThreadID can be used to calculate beginning and end
max span duration:

index=main sourcetype=log4j | transaction maxspan=4h threadId |timechart span=6h max(duration) as MAX, mean(duration) as mean, min(duration) as min

Memory remaining:

index=main sourcetype=log4j perfType=MEMORY |eval mem_used_pc=round((mem_used/mem_total)*100) | eval mem_remaining_pc=(100-mem_used_pc)|timechart span=15m avg(mem_used_pc) as mem_used avg(mem_remaining_pc) as mem_remaining


Counts exceeding access to DB:
index=main sourcetype=log4j perfType="DB" | eval threshold=con_total/100*70 |where con_used>=threshold | timechart span=4h count(con_used) as CountOverThreshold

make a dial / rangemap
index=main sourcetype=access_combined | stats dc(JSESSIONID) as count| rangemap field=count low=0-1 elevated=2-5 default=severe

Session analysis, then set time period!

index=main sourcetype="access_combined" | transaction JSESSIONID|stats avg(duration) AS avg_session_time

Average checkout time:

index=main sourcetype="access_combined" | transaction JSESSIONID startswith="GET /home" endswith="checkout"|stats avg(duration) as avg_checkout_time

maxevents look at 300, maxpause to clarify whether it is same or different session, maxspan is looking at most 30m conversion window

index=main sourcetype="access_combined" | transaction JSESSIONID startswith="GET /home" endswith="checkout" maxpause=30s maxspan=29m maxevents=300|stats avg(duration) as avg_checkout_time

Use time to limit subsearches, use earlier=main search results are matched to later results of subsearch

index=main sourcetype="access_combined" | join JSESSIONID usetime=true earlier=false [search index=main sourcetype=log4j|transaction threadId maxspan=5m|eval JSESSIONID=sessionId] |stats avg(duration) as avg_time
