# splunk

##Some Splunk Syntax:##

sourcetype="access_combined_*" status=200 action=purchase [search sourcetype=access* status=200 action=purchase | top limit=1 clientip|table clientip ]|stats count AS "Total Purch", dc(productId) AS "Total Prods", values(productId) AS "Prod IDs" by clientip | rename clientip as "the big customer"
