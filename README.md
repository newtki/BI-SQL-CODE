# BI-SQL-CODE
SQL codes from various projects
**Findind the top traffic sources**
SELECT 
	utm_source,
    utm_campaign,
    http_referer,
	COUNT(website_session_id) AS Session
FROM website_sessions
WHERE created_at < '2012-04-12'
GROUP BY 
	utm_source,
	utm_campaign,
    http_referer
ORDER BY (Session)DESC

**Traffic Converstion Rates**
SELECT 
	COUNT(DISTINCT(ws.website_session_id)) AS Sessions,
    COUNT(DISTINCT(o.order_id)) AS Orders,
    COUNT(DISTINCT(o.order_id))/COUNT(DISTINCT(ws.website_session_id)) AS session_to_orders_conv
FROM website_sessions ws
	LEFT JOIN orders o
		ON ws.user_id = o.user_id
WHERE ws.created_at < '2012-04-12'
	AND utm_source = 'gsearch'
	AND utm_campaign = 'nonbrand'

**Traffic Source Trending**
SELECT
	MIN(DATE(created_at)),
	COUNT(DISTINCT(website_session_id)) AS Sessions
FROM website_sessions
WHERE created_at < '2012-05-10'
	AND utm_source = 'gsearch'
	AND utm_campaign = 'nonbrand'
GROUP BY YEARWEEK(created_at)

**Traffic Source Bid Optimization**
SELECT 
	ws.device_type,
	COUNT(DISTINCT(ws.website_session_id)) AS Sessions,
    COUNT(DISTINCT(o.order_id)) AS Orders,
    COUNT(DISTINCT(o.order_id))/COUNT(DISTINCT(ws.website_session_id)) AS session_to_orders_conv
FROM website_sessions ws
	LEFT JOIN orders o
		ON  o.website_session_id = ws.website_session_id
WHERE ws.created_at < '2012-05-11'
	AND utm_source = 'gsearch'
	AND utm_campaign = 'nonbrand'
GROUP BY device_type


**Trending with Granular Segments**
SELECT 
	MIN(DATE(created_at)) AS week_start_date,
    SELECT(DISTINCT CASE WHEN device_type = 'Mobile' THEN ws.website_session_id ELSE NULL END) AS session_mobile, 
    SELECT(DISTINCT CASE WHEN device_type = 'Desktop' THEN ws.website_session_id ELSE NULL END) AS session_desktop
FROM website_sessions 
WHERE website_sessions.created_at <'2012-06-09'
	AND website_sessions.created_at > '2012-04-15'
	AND website_sessions.utm_source = 'gsearch'
	AND website_sessions.utm_campaign = 'nonbrand'
GROUP BY YEARWEEK(website_sessions.created_at)


**Finding Top Website Pages**
SELECT 
	pageview_url,
    COUNT(DISTINCT(website_session_id)) AS sessions
FROM website_pageviews
WHERE created_at < '2012-06-09'
GROUP BY pageview_url
ORDER BY sessions DESC


**Identifying Top Entry pages**
	CREATE TEMPORARY TABLE first_pageviews
	SELECT
		website_session_id,
		MIN(website_pageview_id) AS min_pageview_id
	FROM website_pageviews
	WHERE created_at < '2012-06-12'
	GROUP BY 
		website_session_id
*

SELECT 
	wp.pageview_url AS landing_page,
    COUNT(fp.website_session_id) as min_pageviews
FROM first_pageviews fp
	LEFT JOIN website_pageviews wp
		ON  wp.website_pageview_id = fp.min_pageview_id
WHERE wp.created_at < '2012-06-09'
GROUP BY 
    wp.pageview_url	








**CALCULATING BOUNCE RATES**
#STEP 1: finding the first website_pageview_id for relevant sessions
#STEP 2: identifying landing page of each session
#STEP 3: counting pageviews for each session, to identify "bounces"
#STEP 4: summarizing by counting total sessions and bounced sessions


#STEP 1: finding the first website_pageview_id for relevant sessions
	#Find the intital websitepage view id for the website sessionid.
CREATE TEMPORARY TABLE first_pageviews
SELECT 
	MIN(website_pageview_id) AS min_pageviews,
	website_session_id
FROM mavenfuzzyfactory.website_pageviews
GROUP BY website_session_id;

#STEP 2: identifying pageview URL of each session where landing page is home
CREATE TEMPORARY TABLE session_with_landing_page
SELECT
	first_pageviews.min_pageviews,
	first_pageviews.website_session_id,
	website_pageviews.pageview_url AS landing_page
FROM first_pageviews
	LEFT JOIN website_pageviews
		ON first_pageviews.min_pageviews = website_pageviews.website_pageview_id
WHERE pageview_url = '/home';

#STEP 3: Count the pageviews for each session, to identify "bounces" (where count of pages is one)
CREATE TEMPORARY TABLE bounced_sessions
SELECT
	session_with_landing_page.landing_page,
    session_with_landing_page.website_session_id,
    COUNT(website_pageviews.website_session_id) AS count_of_pages_viewed
FROM session_with_landing_page
	JOIN website_pageviews
		ON session_with_landing_page.website_session_id = website_pageviews.website_session_id
GROUP BY
	session_with_landing_page.landing_page,
    session_with_landing_page.website_session_id
HAVING COUNT(website_pageviews.website_session_id) = '1';

#STEP 4: summarizing by counting total sessions and bounced sessions
SELECT
    COUNT(DISTINCT sessions_w_landing_page.website_session_id) AS sessions,
    COUNT(DISTINCT bounced_sessions.website_session_id) AS bounced_sessions,
    COUNT(DISTINCT bounced_sessions.website_session_id) / COUNT(DISTINCT sessions_w_landing_page.website_session_id) AS bounce_rate
FROM sessions_w_landing_page
	LEFT JOIN bounced_sessions
		ON sessions_w_landing_page.website_session_id = bounced_sessions.website_session_id;


**ANALYZING LANDING PAGE TEST**
#STEP 0: Find out when the new page /lander launched ("/lander-1")
#STEP 1: Finding the first website_pageview_id for relevant sessions from step 0 and instructions
#STEP 2: Identifying landing page of each session
#STEP 3: Counting pageviews for each session, to identify "bounces"
#STEP 4: Summarizing by counting total sessions and bounced sessions, by landing page

#STEP 0: find out when the new page /lander launched ("/lander-1")
SELECT
	MIN(created_at) AS first_created_at,
    MIN(website_pageview_id) AS first_pageview_id # Distinct PageviewID
FROM website_pageviews
WHERE pageview_url = '/lander-1'; #First time lander one was displayed on the website

#STEP 1: finding the first website_pageview_id for relevant sessions 
CREATE TEMPORARY TABLE first_pageview_lander1
SELECT
	website_pageviews.website_session_id,
    MIN(website_pageviews.website_pageview_id) AS min_pageview_id
FROM website_pageviews
	INNER JOIN website_sessions
		ON website_pageviews.website_session_id = website_sessions.website_session_id
        AND website_pageviews.created_at < '2012-07-18' #as per assignment
        AND website_pageviews.website_pageview_id > 23504 #as per STEP 0
        AND website_sessions.utm_source = 'gsearch'
        AND website_sessions.utm_campaign = 'nonbrand'
GROUP BY
	website_pageviews.website_session_id;

#STEP 2: identifying landing page of each session
CREATE TEMPORARY TABLE sessions_w_landing_page_lander1
SELECT
	first_pageview_lander1.website_session_id,
    website_pageviews.pageview_url AS landing_page
FROM first_pageview_lander1
	LEFT JOIN website_pageviews
		ON first_pageview_lander1.min_pageview_id = website_pageviews.website_pageview_id
WHERE website_pageviews.pageview_url IN ('/home', '/lander-1');

#STEP 3: counting pageviews for each session, to identify "bounces"
CREATE TEMPORARY TABLE bounced_sessions_lander1
SELECT
	sessions_w_landing_page_lander1.website_session_id,
    sessions_w_landing_page_lander1.landing_page,
    COUNT(website_pageviews.website_pageview_id) AS count_of_pages_viewed
FROM sessions_w_landing_page_lander1
	LEFT JOIN website_pageviews
		ON website_pageviews.website_session_id = sessions_w_landing_page_lander1.website_session_id
GROUP BY
	sessions_w_landing_page_lander1.website_session_id,
    sessions_w_landing_page_lander1.landing_page
HAVING
	COUNT(website_pageviews.website_pageview_id) = 1;

#STEP 4: summarizing by counting total sessions and bounced sessions, by landing page
SELECT
	sessions_w_landing_page_lander1.landing_page,
    COUNT(DISTINCT sessions_w_landing_page_lander1.website_session_id) AS sessions,
    COUNT(DISTINCT bounced_sessions_lander1.website_session_id) AS bounced_sessions,
    COUNT(DISTINCT bounced_sessions_lander1.website_session_id) / COUNT(DISTINCT sessions_w_landing_page_lander1.website_session_id) AS bounce_rate
FROM sessions_w_landing_page_lander1
	LEFT JOIN bounced_sessions_lander1
		ON sessions_w_landing_page_lander1.website_session_id = bounced_sessions_lander1.website_session_id
GROUP BY
	sessions_w_landing_page_lander1.landing_page;
