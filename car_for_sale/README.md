Welcome to Rails
================

```ruby
class WeblogController < ActionController::Base
 def destroy
 	@weblog = Weblog.find(params[:id])
 	@weblog.destroy
  logger.info("#{Time.now} Destroyed Weblog ID ##{@weblog.id}!")
 end
end
```
REAL CASE EXAMPLE
=================
   - We want to simulate Google Insight's search form for every available categories and 
   - We will do daily scrape and will use the same query for each day
   - We want to use proxies and switch between them (and proxies is saved in text file)
   - Save search's results into DB


```ruby
#custom library
require 'ginsight'
require 'ginsight-db'
require 'my_proxy' 

#db connection
ActiveRecord::Base.establish_connection(YAML.load_file(File.dirname(__FILE__) + '/../config/database.yaml'))

#load proxies
proxies = open(File.expand_path(File.dirname(__FILE__)) + '/../bin/proxies.txt').map { |line| line.chomp }
proxy = MyProxy.new(proxies)

#new agent
agent = Ginsight.new{|a| a.user_agent_alias = "Linux Mozilla"}

#set proxy
agent.set_proxy(proxy.proxy_host, proxy.proxy_port)

#get all categories from DB
Category.all.each do |c| 
  # prepare for query string
	# we will use same query for each categories
	data = { 
		:date => [Time.now.month, Time.now.year].join("/") + " 1m", 
		:geo     => 'US', 
		:cmpt    => 'q', 
		:content => '1',  
		:cat     => c.cat_id
	}

	# multiple scrapes with same query and same category and also same date is not allowed
	next if !Query.today_query(data).nil?

  # it's like "get" and "parse"
	begin
		agent.scrape(data)
	rescue StandardError=> e
		# if error
		puts e.message
		if e.message =~ /(connection (refused|timed out)|No route to host|too many connection resets)/i
			# delete current proxy from list
			proxy.delete
			# set new proxy
			agent.set_proxy(proxy.proxy_host, proxy.proxy_port)
			# set new special cookie
			agent.custom_cookie = agent.custom_random
		end

		if e.respond_to?('response_code')
			puts e.response_code
			if e.response_code =~ /^(4|5)/
				proxy.delete
				agent.set_proxy(proxy.proxy_host, proxy.proxy_port)
				agent.custom_cookie = agent.custom_random
			end
		end
		redo
	end
	
	if agent.limit_reached?
		puts "You have reached your quota limit. Please try again later"

		proxy.delete
		agent.set_proxy(proxy.proxy_host, proxy.proxy_port)
		agent.custom_cookie = agent.custom_random
		
		redo;
	end

	last_query = Query.create(data)
	
	rising_result = agent.rising_result

	# save result to DB
	rising_result.each do |k|
		k[:query_id] = last_query.id
		Result.create(k)
		puts "New result saved"
	end

	sleep 3;
end
```

TABLES
======
Soon




<table>
  <tr>
    <th>ID</th><th>Name</th><th>Rank</th>
  </tr>
  <tr>
    <td>1</td><td>Tom Preston-Werner</td><td>Awesome</td>
  </tr>
  <tr>
    <td>2</td><td>Albert Einstein</td><td>Nearly as awesome</td>
  </tr>
</table>

DB TABLES & RELATIONSHIPS
=========================

Schema
------

<pre>
      Table "public.categories"
      Column     |            Type             | Modifiers 
      -----------+-----------------------------+-----------
  --- id         | character varying(70)       | not null
  |	  name       | text                        | not null
  |	  created_at | timestamp without time zone | 
  |	  updated_at | timestamp without time zone |
  |
  |
  |     Table "public.queries"
  |     Column     |            Type             |                      Modifiers                       
  |	    -----------+-----------------------------+------------------------------------------------------
  |	--- id         | integer                     | not null default nextval('queries_id_seq'::regclass)
  --|->	cat        | character varying(70)       | 
    |   date       | character varying(50)       | 
    |   geo        | character varying(20)       | not null
    |   cmpt       | character varying(10)       | not null
    |   content    | character varying(5)        | not null
    |   created_at | timestamp without time zone | 
    |   updated_at | timestamp without time zone | 
    |   hl         | character varying(10)       | 
    |
    |
    |
    |   Table "public.results"
    |   Column     |            Type             |                      Modifiers                       
    |   -----------+-----------------------------+------------------------------------------------------
    |   id         | integer                     | not null default nextval('results_id_seq'::regclass)
    --> query_id   | integer                     | 
        ranking    | integer                     | not null
        result     | text                        | not null
        percentage | character varying(50)       | 
        created_at | timestamp without time zone | 
        updated_at | timestamp without time zone |
</pre>

Examples
--------

Categories
<table>
	<tr>
		<td>id</td>
		<td>name</td>
		<td>created_at</td>
		<td>updated_at</td>
	</tr>
	<tr>
		<td>0-3</td>
		<td>Arts & Entertainment</td>
		<td>2012-08-26 01:27:49.126124</td>
		<td>2012-08-26 01:27:49.126124</td>
	</tr>
	<tr>
		<td>0-3-184</td>
		<td>Celebrities & Entertainment News</td>
		<td>2012-08-26 01:27:49.185775</td>
		<td>2012-08-26 01:27:49.185775</td>
	</tr>
</table>

Queries
<table>
	<tr>
		<td>id</td>
		<td>cat</td>
		<td>date</td>
		<td>geo</td>
		<td>cmpt</td>
		<td>content</td>
		<td>created_at</td>
		<td>updated_at</td>
		<td>hl</td>
	</tr>
	<tr>
		<td>22822</td>
		<td>0-3</td>
		<td>8/2012 1m</td>
		<td>US</td>
		<td>q</td>
		<td>1</td>
		<td>2012-08-26 01:50:39.017567</td>
		<td>2012-08-26 01:50:39.017567</td>
		<td>en</td>
	</tr>
	<tr>
		<td>24619</td>
		<td>0-3</td>
		<td>8/2012 1m</td>
		<td>US</td>
		<td>q</td>
		<td>1</td>]
		<td>2012-08-27 00:10:10.759103</td>
		<td>2012-08-27 00:10:10.759103</td>
		<td>en</td>
	</tr>
</table>

Results
<table>
	<tr>
		<td>id</td>
		<td>query_id</td>
		<td>ranking</td>
		<td>result</td>
		<td>percentage</td>
		<td>created_at</td>
		<td>updated_at</td>
	</tr>
	<tr>
		<td>38308</td>
		<td>22822</td>
		<td>1</td>
		<td>miley cyrus haircut</td>
		<td>Breakout</td>
		<td>2012-08-26 01:50:39.122775</td>
		<td>2012-08-26 01:50:39.122775</td>
	</tr>
	<tr>
		<td>38309</td>
		<td>22822</td>
		<td>2</td>
		<td>tony scott</td>
		<td>Breakout</td>
		<td>2012-08-26 01:50:39.155391</td>
		<td>2012-08-26 01:50:39.155391</td>
	</tr>
</table>


