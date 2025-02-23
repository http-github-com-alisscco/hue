---
title: REST API for sending SQL queries and Browsing files
author: Hue Team
type: post
date: 2020-09-14T00:00:00+00:00
url: /blog/rest-api-execute-sql-queries-browse-files/
sf_thumbnail_type:
  - none
sf_thumbnail_link_type:
  - link_to_post
sf_detail_type:
  - none
sf_page_title:
  - 1
sf_page_title_style:
  - standard
sf_no_breadcrumbs:
  - 1
sf_page_title_bg:
  - none
sf_page_title_text_style:
  - light
sf_background_image_size:
  - cover
sf_social_sharing:
  - 1
sf_related_articles:
  - 1
sf_sidebar_config:
  - left-sidebar
sf_left_sidebar:
  - Sidebar-2
sf_right_sidebar:
  - Sidebar-1
sf_caption_position:
  - caption-right
sf_remove_promo_bar:
  - 1
ampforwp-amp-on-off:
  - default
categories:
  - Version 4
#  - Version 4.8
  - Development

---
Hi Data App Builders,

Are you looking at executing some SQL queries of Browsing S3 files programatically? (so that it can be automated for example)

![Curling Hue API](https://cdn.gethue.com/uploads/2020/09/hue-curl.png)

The Hue development flow continues to mature ([Docker Quick Start](https://gethue.com/quickstart-hue-in-docker/), [improved CI](https://gethue.com/automated-checking-python-style-and-title-format-of-git-commits-continuous-integration/), shareable [Web Components](https://docs.gethue.com/developer/components/)...) and is now getting more help on how to reuse its API.

## Concept

The REST API is not properly public yet and can (will) be simplified in the current work in progress [HUE-1450](https://issues.cloudera.org/browse/HUE-1450).

Hue is Ajax based and has a REST API used by the browser to communicate (e.g. submit an SQL query, list some S3 files in a buck...) with the Hue server which will then perform the operation with the remote services. Currently this API is private and subject to change but can already be reused for prove of concepts until the proper public API is released.

In general we want to authenticate with a username and password, retrieve a cookie and [CSRF](https://docs.djangoproject.com/en/3.1/ref/csrf/) token and provide them in the follow-up calls so that Hue knows who we are and won't block you.

The [API documentation](https://docs.gethue.com/developer/api/) has been refreshed and list new endpoints.

## Login

You would need to GET the `/accounts/login` page to get the CSRF token and POST it back along `username` and `password` and reuse the `sessionid` cookie and `CSRF` token in the next communication calls.

Here we just ask for the page in order to get the tokens in the headers:

    curl -i -X GET https://demo.gethue.com/hue/accounts/login/?fromModal=true -o /dev/null -D -
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
      0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0HTTP/2 200
    server: nginx/1.19.0
    date: Mon, 14 Sep 2020 20:43:47 GMT
    content-type: text/html; charset=utf-8
    content-length: 115010
    vary: Accept-Encoding
    set-cookie: hue-balancer=1600116228.241.36.408868; Expires=Wed, 16-Sep-20 20:43:47 GMT; Max-Age=172800; Path=/; Secure; HttpOnly
    x-xss-protection: 1; mode=block
    content-security-policy: script-src 'self' 'unsafe-inline' 'unsafe-eval' *.google-analytics.com *.doubleclick.net data:;img-src 'self' *.google-analytics.com *.doubleclick.net http://*.tile.osm.org *.tile.osm.org *.gstatic.com data:;style-src 'self' 'unsafe-inline' fonts.googleapis.com;connect-src 'self';frame-src *;child-src 'self' data: *.vimeo.com;object-src 'none'
    x-content-type-options: nosniff
    content-language: en-us
    vary: Cookie, Accept-Language
    etag: "c5801a9442bf6e3aaecb597b1640a119"
    x-frame-options: SAMEORIGIN
    set-cookie: csrftoken=XUFgN1WPZNlaJtBeBDtBvwzrOFqRXIaMlNJv4mdvsS2bIE2Lb8LRmCh5cPUBnBdk; expires=Mon, 13-Sep-2021 20:43:47 GMT; httponly; Max-Age=31449600; Path=/
    set-cookie: sessionid=9cdltfee1q1zmt8b7slsjcomtxzgvgfz; expires=Mon, 14-Sep-2020 21:43:47 GMT; httponly; Max-Age=3600; Path=/
    set-cookie: ROUTEID=; expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
    strict-transport-security: max-age=15724800; includeSubDomains

    100  112k  100  112k    0     0   218k      0 --:--:-- --:--:-- --:--:--  218k

It is important to note the `csrftoken` and `sessionid` values from the response:

    set-cookie: csrftoken=XUFgN1WPZNlaJtBeBDtBvwzrOFqRXIaMlNJv4mdvsS2bIE2Lb8LRmCh5cPUBnBdk; expires=Mon, 13-Sep-2021 20:43:47 GMT; httponly; Max-Age=31449600; Path=/
    set-cookie: sessionid=9cdltfee1q1zmt8b7slsjcomtxzgvgfz; expires=Mon, 14-Sep-2020 21:43:47 GMT; httponly; Max-Age=3600; Path=/

And then we can authenticate for real with the username and password credendials `demo` / `demo`:

    curl -i -X POST https://demo.gethue.com/hue/accounts/login/?fromModal=true -d 'username=demo&password=demo' -o /dev/null -D - --cookie "csrftoken=Jfd8BoJGQVYLZsBJkcw1TCXPPgkKHMtdDmRx7n3KGMQevXmmHTpn3pcnoLkzo9mD;sessionid=c4j5ewhvu1dm9f8jojbh049zyitse72j" -H "X-CSRFToken: Jfd8BoJGQVYLZsBJkcw1TCXPPgkKHMtdDmRx7n3KGMQevXmmHTpn3pcnoLkzo9mD"
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
      0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0HTTP/2 200
    server: nginx/1.19.0
    date: Mon, 14 Sep 2020 20:53:59 GMT
    content-type: application/json
    content-length: 14
    set-cookie: hue-balancer=1600116840.493.37.925584; Expires=Wed, 16-Sep-20 20:53:59 GMT; Max-Age=172800; Path=/; Secure; HttpOnly
    x-xss-protection: 1; mode=block
    content-security-policy: script-src 'self' 'unsafe-inline' 'unsafe-eval' *.google-analytics.com *.doubleclick.net data:;img-src 'self' *.google-analytics.com *.doubleclick.net http://*.tile.osm.org *.tile.osm.org *.gstatic.com data:;style-src 'self' 'unsafe-inline' fonts.googleapis.com;connect-src 'self';frame-src *;child-src 'self' data: *.vimeo.com;object-src 'none'
    x-content-type-options: nosniff
    content-language: en-us
    vary: Cookie, Accept-Language
    x-frame-options: SAMEORIGIN
    set-cookie: csrftoken=C5vyqWODpIZi1oCyeIWNrrdMimejy4KXDI3R22qq4Bc6emCzLRakqQ1KYwNbOH0q; expires=Mon, 13-Sep-2021 20:53:59 GMT; httponly; Max-Age=31449600; Path=/
    set-cookie: sessionid=0ein2dkwb1g5fz29o2d75vx9t7uv33os; expires=Mon, 14-Sep-2020 21:53:59 GMT; httponly; Max-Age=3600; Path=/
    strict-transport-security: max-age=15724800; includeSubDomains

    100    41  100    14  100    27     38     74 --:--:-- --:--:-- --:--:--   112

This time the `cookie` and `CSRF` tokens are the one that we will need for performing actual actions with the API. Here for example we list all the databases of the `hive` connector:

    curl -X POST https://demo.gethue.com/notebook/api/autocomplete/ --data 'snippet={"type":"hive"}' --cookie "csrftoken=C5vyqWODpIZi1oCyeIWNrrdMimejy4KXDI3R22qq4Bc6emCzLRakqQ1KYwNbOH0q;sessionid=0ein2dkwb1g5fz29o2d75vx9t7uv33os" -H "X-CSRFToken: C5vyqWODpIZi1oCyeIWNrrdMimejy4KXDI3R22qq4Bc6emCzLRakqQ1KYwNbOH0q"

    {"status": 0, "databases": ["a0817", "arunso", "ath", "athlete", "beingdatum_db", "bharath_practice", "chungu", "darth", "default", "demo", "diwakar", "emp", "hadooptest", "hebe", "hello", "hivedataset", "hivemap", "hivetesting", "hr_db", "icedb", "lib", "libdemo", "lti", "m", "m1", "movie", "movie1", "movielens", "mscda", "my_database", "mydb", "mydemo", "noo", "prizesh", "rajeev", "ram", "retail", "ruban", "sept9", "sept9_2020", "student", "test", "test21", "test123", "ticktick", "userdb", "vidu"]}


**Note** There is currently no error on bad authentication but instead a 302 redirect to the login page, e.g.:

    [12/Sep/2020 10:12:44 -0700] middleware   INFO     Redirecting to login page: /hue/useradmin/users/edit/romain
    [12/Sep/2020 10:12:44 -0700] access       INFO     127.0.0.1 -anon- - "POST /hue/useradmin/users/edit/romain HTTP/1.1" - (mem: 169mb)-- login redirection
    [12/Sep/2020 10:12:44 -0700] access       INFO     127.0.0.1 -anon- - "POST /hue/useradmin/users/edit/romain HTTP/1.1" returned in 0ms 302 0 (mem: 169mb)

**Note** The public API should be simpler with only one POST call and will return back only one token

## SQL Querying

Now that we are logged-in, here is how to execute a `SHOW TABLES` SQL query via the `hive` connector. You could repeat the steps with any query you want, e.g. `SELECT * FROM web_logs LIMIT 100`.

Until Editor v2 is out [HUE-8768](https://issues.cloudera.org/browse/HUE-8768), the API is pretty complicated but still functional:

For a `SHOW TABLES`, first we send the query statement:

    curl -X POST https://demo.gethue.com/notebook/api/execute/hive --data 'executable={"statement":"SHOW TABLES","database":"default"}&notebook={"type":"query","snippets":[{"id":1,"statement_raw":"SHOW TABLES","type":"hive","variables":[]}],"name":"","isSaved":false,"sessions":[]}&snippet={"id":1,"type":"hive","result":{},"statement":"SHOW TABLES","properties":{}}' --cookie "csrftoken=lFpBt6uaa8isgjIthEFZdUbofKPI7wJaXpWS0q54YKORs8zWvKgvrKzwUnTjsFt3;sessionid=5msr9s45o4emem1zt009kw64ni6uso3e" -H "X-CSRFToken: lFpBt6uaa8isgjIthEFZdUbofKPI7wJaXpWS0q54YKORs8zWvKgvrKzwUnTjsFt3"

    {"status": 0, "history_id": 17880, "handle": {"statement_id": 0, "session_type": "hive", "has_more_statements": false, "guid": "EUI32vrfTkSOBXET6Eaa+A==\n", "previous_statement_hash": "3070952e55d733fb5bef249277fb8674989e40b6f86c5cc8b39cc415", "log_context": null, "statements_count": 1, "end": {"column": 10, "row": 0}, "session_id": 63, "start": {"column": 0, "row": 0}, "secret": "RuiF0LEkRn+Yok/gjXWSqg==\n", "has_result_set": true, "session_guid": "c845bb7688dca140:859a5024fb284ba2", "statement": "SHOW TABLES", "operation_type": 0, "modified_row_count": null}, "history_uuid": "63ce87ba-ca0f-4653-8aeb-e9f5c1781b78"}

Then check the operation status until it is available for fetching its result:

    curl -X POST https://demo.gethue.com/notebook/api/check_status --data 'notebook={"type":"hive"}&snippet={"history_id": 17886,"type":"hive","result":{"handle":{"guid": "0J6PwGcSQaCJjagzYUBHzA==\n","secret": "uiP3IS4fR/mxkLJER5wRCg==\n","has_result_set": true}},"status":""}' --cookie "csrftoken=lFpBt6uaa8isgjIthEFZdUbofKPI7wJaXpWS0q54YKORs8zWvKgvrKzwUnTjsFt3;sessionid=5msr9s45o4emem1zt009kw64ni6uso3e" -H "X-CSRFToken: lFpBt6uaa8isgjIthEFZdUbofKPI7wJaXpWS0q54YKORs8zWvKgvrKzwUnTjsFt3"

    {"status": 0, "query_status": {"status": "available", "has_result_set": true}}

And now ask for the resultset of the statement:

    curl -X POST https://demo.gethue.com/notebook/api/fetch_result_data --data 'notebook={"type":"hive"}&snippet={"history_id": 17886,"type":"hive","result":{"handle":{"guid": "0J6PwGcSQaCJjagzYUBHzA==\n","secret": "uiP3IS4fR/mxkLJER5wRCg==\n","has_result_set": true}},"status":""}' --cookie "csrftoken=lFpBt6uaa8isgjIthEFZdUbofKPI7wJaXpWS0q54YKORs8zWvKgvrKzwUnTjsFt3;sessionid=5msr9s45o4emem1zt009kw64ni6uso3e" -H "X-CSRFToken: lFpBt6uaa8isgjIthEFZdUbofKPI7wJaXpWS0q54YKORs8zWvKgvrKzwUnTjsFt3"

    {"status": 0, "result": {"has_more": true, "type": "table", "meta": [{"comment": "from deserializer", "type": "STRING_TYPE", "name": "tab_name"}], "data": [["adavi"], ["adavi1"], ["adavi2"], ["ambs_feed"], ["apx_adv_deduction_data_process_total"], ["avro_table"], ["avro_table1"], ["bb"], ["bharath_info1"], ["bucknew"], ["bucknew1"], ["chungu"], ["cricket3"], ["cricket4"], ["cricket5_view"], ["cricketer"], ["cricketer_view"], ["cricketer_view1"], ["demo1"], ["demo12345"], ["dummy"], ["embedded"], ["emp"], ["emp1_sept9"], ["emp_details"], ["emp_sept"], ["emp_tbl1"], ["emp_tbl2"], ["empdtls"], ["empdtls_ext"], ["empdtls_ext_v2"], ["employee"], ["employee1"], ["employee_ins"], ["empppp"], ["events"], ["final"], ["flight_data"], ["gopalbhar"], ["guruhive_internaltable"], ["hell"], ["info1"], ["lost_messages"], ["mnewmyak"], ["mortality"], ["mscda"], ["myak"], ["mysample"], ["mysample1"], ["mysample2"], ["network"], ["ods_t_exch_recv_rel_wfz_stat_szy"], ["olympicdata"], ["p_table"], ["partition_cricket"], ["partitioned_user"], ["s"], ["sample"], ["sample_07"], ["sample_08"], ["score"], ["stg_t_exch_recv_rel_wfz_stat_szy"], ["stocks"], ["students"], ["studentscores"], ["studentscores2"], ["t1"], ["table_name"], ["tablex"], ["tabley"], ["temp"], ["test1"], ["test2"], ["test21"], ["test_info"], ["topage"], ["txnrecords"], ["u_data"], ["udata"], ["user_session"], ["user_test"], ["v_empdtls"], ["v_empdtls_ext"], ["v_empdtls_ext_v2"], ["web_logs"]], "isEscaped": true}}

And if we wanted to get the execution log for this statement:

    curl -X POST https://demo.gethue.com/notebook/api/get_logs --data 'notebook={"type":"hive","sessions":[]}&snippet={"history_id": 17886,"type":"hive","result":{"handle":{"guid": "0J6PwGcSQaCJjagzYUBHzA==\n","secret": "uiP3IS4fR/mxkLJER5wRCg==\n","has_result_set": true}},"status":"","properties":{},"sessions":[]}' --cookie "csrftoken=lFpBt6uaa8isgjIthEFZdUbofKPI7wJaXpWS0q54YKORs8zWvKgvrKzwUnTjsFt3;sessionid=5msr9s45o4emem1zt009kw64ni6uso3e" -H "X-CSRFToken: lFpBt6uaa8isgjIthEFZdUbofKPI7wJaXpWS0q54YKORs8zWvKgvrKzwUnTjsFt3"

    {"status": 0, "progress": 5, "jobs": [], "logs": "", "isFullLogs": false}

## File Browsing

Hue's [File Browser](https://docs.gethue.com/user/browsing/#data) offer uploads, downloads and listing of data in HDFS, S3, ADLS storages.

Here is how to list the content of a path, here the S3 bucket `S3A://gethue-demo`:

    curl -X GET "https://demo.gethue.com/filebrowser/view=S3A://gethue-demo?pagesize=45&pagenum=1&filter=&sortby=name&descending=false&format=json" --cookie "csrftoken=oT8C5cQCbmpuoKcUZ2YaxybfLhtRShEO9UcvRWetx4HVatLuf6qicgJnbEHxfJNI;sessionid=nkblu68xfofabfsjctdwseaubfbkiwlg" -H "X-CSRFToken: oT8C5cQCbmpuoKcUZ2YaxybfLhtRShEO9UcvRWetx4HVatLuf6qicgJnbEHxfJNI"

    {
      ...........
      "files": [
      {
      "humansize": "0\u00a0bytes",
      "url": "/filebrowser/view=s3a%3A%2F%2Fdemo-hue",
      "stats": {
      "size": 0,
      "aclBit": false,
      "group": "",
      "user": "",
      "mtime": null,
      "path": "s3a://demo-hue",
      "atime": null,
      "mode": 16895
      },
      "name": "demo-hue",
      "mtime": "",
      "rwx": "drwxrwxrwx",
      "path": "s3a://demo-hue",
      "is_sentry_managed": false,
      "type": "dir",
      "mode": "40777"
      },
      {
      "humansize": "0\u00a0bytes",
      "url": "/filebrowser/view=S3A%3A%2F%2F",
      "stats": {
      "size": 0,
      "aclBit": false,
      "group": "",
      "user": "",
      "mtime": null,
      "path": "S3A://",
      "atime": null,
      "mode": 16895
      },
      "name": ".",
      "mtime": "",
      "rwx": "drwxrwxrwx",
      "path": "S3A://",
      "is_sentry_managed": false,
      "type": "dir",
      "mode": "40777"
      }
      ],
      ...........
    }

How to get the file content and its metadata. Here with the public file of demo.gethue.com [s3a://demo-hue/web_log_data/index_data.csv](https://demo.gethue.com/hue/filebrowser/view=s3a%3A%2F%2Fdemo-hue%2Fweb_log_data%2Findex_data.csv):

**Note** It needs the `XMLHttpRequest` header to return the data in json:

    curl  -X GET "https://demo.gethue.com/filebrowser/view=s3a://demo-hue/web_log_data/index_data.csv?offset=0&length=204800&compression=none&mode=text" --cookie "csrftoken=oT8C5cQCbmpuoKcUZ2YaxybfLhtRShEO9UcvRWetx4HVatLuf6qicgJnbEHxfJNI;sessionid=nkblu68xfofabfsjctdwseaubfbkiwlg" -H "X-CSRFToken: oT8C5cQCbmpuoKcUZ2YaxybfLhtRShEO9UcvRWetx4HVatLuf6qicgJnbEHxfJNI" -H "X-requested-with: XMLHttpRequest"

    {
      "show_download_button": true,
      "is_embeddable": false,
      "editable": false,
      "mtime": "October 31, 2016 03:34 PM",
      "rwx": "-rw-rw-rw-",
      "path": "s3a://demo-hue/web_log_data/index_data.csv",
      "stats": {
      "size": 6199593,
      "aclBit": false,
      ...............
      "contents": "code,protocol,request,app,user_agent_major,region_code,country_code,id,city,subapp,latitude,method,client_ip,  user_agent_family,bytes,referer,country_name,extension,url,os_major,longitude,device_family,record,user_agent,time,os_family,country_code3
        200,HTTP/1.1,GET /metastore/table/default/sample_07 HTTP/1.1,metastore,,00,SG,8836e6ce-9a21-449f-a372-9e57641389b3,Singapore,table,1.2931000000000097,GET,128.199.234.236,Other,1041,-,Singapore,,/metastore/table/default/sample_07,,103.85579999999999,Other,"demo.gethue.com:80 128.199.234.236 - - [04/May/2014:06:35:49 +0000] ""GET /metastore/table/default/sample_07 HTTP/1.1"" 200 1041 ""-"" ""Mozilla/5.0 (compatible; phpservermon/3.0.1; +http://www.phpservermonitor.org)""
        ",Mozilla/5.0 (compatible; phpservermon/3.0.1; +http://www.phpservermonitor.org),2014-05-04T06:35:49Z,Other,SGP
        200,HTTP/1.1,GET /metastore/table/default/sample_07 HTTP/1.1,metastore,,00,SG,6ddf6e38-7b83-423c-8873-39842dca2dbb,Singapore,table,1.2931000000000097,GET,128.199.234.236,Other,1041,-,Singapore,,/metastore/table/default/sample_07,,103.85579999999999,Other,"demo.gethue.com:80 128.199.234.236 - - [04/May/2014:06:35:50 +0000] ""GET /metastore/table/default/sample_07 HTTP/1.1"" 200 1041 ""-"" ""Mozilla/5.0 (compatible; phpservermon/3.0.1; +http://www.phpservermonitor.org)""
        ",Mozilla/5.0 (compatible; phpservermon/3.0.1; +http://www.phpservermonitor.org),2014-05-04T06:35:50Z,Other,SGP
      ...............
    }


And that's it! The next iterations will make the API simpler with less parameters and versioning.

Any feedback or question? Feel free to comment here or on the <a href="https://discourse.gethue.com/">Forum</a> and <a href="https://docs.gethue.com/quickstart/">quick start</a> SQL querying!


Onwards!

Romain from the Hue Team
