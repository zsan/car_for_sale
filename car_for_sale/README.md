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

